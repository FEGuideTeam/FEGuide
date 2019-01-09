> 面试中经常会遇到的手写代码系列

### 函数防抖(debounce)

概念:在事件被触发 n 秒后再执行回调，如果在这 n 秒内又被触发，则重新计时。

```js
function debounce(fn, wait) {
  var timer = null;
  return function() {
    var context = this;
    var args = arguments;
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }
    timer = setTimeout(function() {
      fn.apply(context, args);
    }, wait);
  };
}
```

### 函数节流 throttle

概念： 规定一个单位时间，在这个单位时间内，只能有一次触发事件的回调函数执行，如果在同一个单位时间内某事件被触发多次，只有一次能生效。

```js
function throttle(fn, gapTime) {
  let _lastTime = null;

  return function() {
    let _nowTime = +new Date();
    if (_nowTime - _lastTime > gapTime || !_lastTime) {
      fn();
      _lastTime = _nowTime;
    }
  };
}
```

### 手写 promise

[剖析 Promise 内部结构，一步一步实现一个完整的、能通过所有 Test case 的 Promise 类 ](https://github.com/xieranmaya/blog/issues/3)

```js
try {
  module.exports = Promise;
} catch (e) {}

function Promise(executor) {
  var self = this;

  self.status = "pending";
  self.onResolvedCallback = [];
  self.onRejectedCallback = [];

  function resolve(value) {
    if (value instanceof Promise) {
      return value.then(resolve, reject);
    }
    setTimeout(function() {
      // 异步执行所有的回调函数
      if (self.status === "pending") {
        self.status = "resolved";
        self.data = value;
        for (var i = 0; i < self.onResolvedCallback.length; i++) {
          self.onResolvedCallback[i](value);
        }
      }
    });
  }

  function reject(reason) {
    setTimeout(function() {
      // 异步执行所有的回调函数
      if (self.status === "pending") {
        self.status = "rejected";
        self.data = reason;
        for (var i = 0; i < self.onRejectedCallback.length; i++) {
          self.onRejectedCallback[i](reason);
        }
      }
    });
  }

  try {
    executor(resolve, reject);
  } catch (reason) {
    reject(reason);
  }
}

function resolvePromise(promise2, x, resolve, reject) {
  var then;
  var thenCalledOrThrow = false;

  if (promise2 === x) {
    return reject(new TypeError("Chaining cycle detected for promise!"));
  }

  if (x instanceof Promise) {
    if (x.status === "pending") {
      //because x could resolved by a Promise Object
      x.then(function(v) {
        resolvePromise(promise2, v, resolve, reject);
      }, reject);
    } else {
      //but if it is resolved, it will never resolved by a Promise Object but a static value;
      x.then(resolve, reject);
    }
    return;
  }

  if (x !== null && (typeof x === "object" || typeof x === "function")) {
    try {
      then = x.then; //because x.then could be a getter
      if (typeof then === "function") {
        then.call(
          x,
          function rs(y) {
            if (thenCalledOrThrow) return;
            thenCalledOrThrow = true;
            return resolvePromise(promise2, y, resolve, reject);
          },
          function rj(r) {
            if (thenCalledOrThrow) return;
            thenCalledOrThrow = true;
            return reject(r);
          }
        );
      } else {
        resolve(x);
      }
    } catch (e) {
      if (thenCalledOrThrow) return;
      thenCalledOrThrow = true;
      return reject(e);
    }
  } else {
    resolve(x);
  }
}

Promise.prototype.then = function(onResolved, onRejected) {
  var self = this;
  var promise2;
  onResolved =
    typeof onResolved === "function"
      ? onResolved
      : function(v) {
          return v;
        };
  onRejected =
    typeof onRejected === "function"
      ? onRejected
      : function(r) {
          throw r;
        };

  if (self.status === "resolved") {
    return (promise2 = new Promise(function(resolve, reject) {
      setTimeout(function() {
        // 异步执行onResolved
        try {
          var x = onResolved(self.data);
          resolvePromise(promise2, x, resolve, reject);
        } catch (reason) {
          reject(reason);
        }
      });
    }));
  }

  if (self.status === "rejected") {
    return (promise2 = new Promise(function(resolve, reject) {
      setTimeout(function() {
        // 异步执行onRejected
        try {
          var x = onRejected(self.data);
          resolvePromise(promise2, x, resolve, reject);
        } catch (reason) {
          reject(reason);
        }
      });
    }));
  }

  if (self.status === "pending") {
    // 这里之所以没有异步执行，是因为这些函数必然会被resolve或reject调用，而resolve或reject函数里的内容已是异步执行，构造函数里的定义
    return (promise2 = new Promise(function(resolve, reject) {
      self.onResolvedCallback.push(function(value) {
        try {
          var x = onResolved(value);
          resolvePromise(promise2, x, resolve, reject);
        } catch (r) {
          reject(r);
        }
      });

      self.onRejectedCallback.push(function(reason) {
        try {
          var x = onRejected(reason);
          resolvePromise(promise2, x, resolve, reject);
        } catch (r) {
          reject(r);
        }
      });
    }));
  }
};

Promise.prototype.catch = function(onRejected) {
  return this.then(null, onRejected);
};

Promise.deferred = Promise.defer = function() {
  var dfd = {};
  dfd.promise = new Promise(function(resolve, reject) {
    dfd.resolve = resolve;
    dfd.reject = reject;
  });
  return dfd;
};
```

### 数组降重

[优雅的数组降维——Javascript 中 apply 方法的妙用](https://www.cnblogs.com/front-end-ralph/p/4871332.html)

1. 朴素的转换,利用 for

```js
function reduceDimension(arr) {
  var reduced = [];
  for (var i = 0; i < arr.length; i++) {
    for (var j = 0; j < arr[i].length; j++) {
      reduced.push(arr[i][j]);
    }
  }
  return reduced;
}
```

此方法思路简单，利用双重循环遍历二维数组中的每个元素并放到新数组中。

2. 利用 concat 转换

```js
function reduceDimension(arr) {
  var reduced = [];
  for (var i = 0; i < arr.length; i++) {
    reduced = reduced.concat(arr[i]);
  }
  return reduced;
}
```

arr 的每一个元素都是一个数组，作为 concat 方法的参数，数组中的每一个子元素又都会被独立插入进新数组。
利用 concat 方法，我们将双重循环简化为了单重循环。

3. 利用 apply 和 concat 转换

```js
function reduceDimension(arr) {
  return Array.prototype.concat.apply([], arr);
}
```

arr 作为 apply 方法的第二个参数，本身是一个数组，数组中的每一个元素（还是数组，即二维数组的第二维）会被作为参数依次传入到 concat 中，效果等同于[].concat([1,2], [3,4], [5,6])。
利用 apply 方法，我们将单重循环优化为了一行代码，很简洁有型有木有啊~

### 数组乱序

[JavaScript 数组乱序](https://github.com/hanzichi/underscore-analysis/issues/15)

Fisher–Yates Shuffle :其实它的思想非常的简单，遍历数组元素，将其与之前的任意元素交换。因为遍历有从前向后和从后往前两种方式，所以该算法大致也有两个版本的实现。

```js
function shuffle(array) {
  var _array = array.concat();

  for (var i = _array.length; i--; ) {
    var j = Math.floor(Math.random() * (i + 1));
    var temp = _array[i];
    _array[i] = _array[j];
    _array[j] = temp;
  }

  return _array;
}
```

underscore 中采用从前往后遍历元素的方式，实现如下：

```js
// Shuffle a collection, using the modern version of the
// [Fisher-Yates shuffle](http://en.wikipedia.org/wiki/Fisher–Yates_shuffle).
_.shuffle = function(obj) {
  var set = isArrayLike(obj) ? obj : _.values(obj);
  var length = set.length;
  var shuffled = Array(length);
  for (var index = 0, rand; index < length; index++) {
    rand = _.random(0, index);
    if (rand !== index) shuffled[index] = shuffled[rand];
    shuffled[rand] = set[index];
  }
  return shuffled;
};
```

### 实现 bind

[bind 方法的兼容实现](https://github.com/hanzichi/underscore-analysis/issues/19)

```js
Function.prototype.bind =
  Function.prototype.bind ||
  function(context) {
    var that = this;
    return function() {
      return that.apply(context, arguments);
    };
  };
```

[underscore 源码](https://github.com/hanzichi/underscore-analysis/blob/master/underscore-1.8.3.js/src/underscore-1.8.3.js#L698-L719%E3%80%82)

```js
// Determines whether to execute a function as a constructor
// or a normal function with the provided arguments
var executeBound = function(
  sourceFunc,
  boundFunc,
  context,
  callingContext,
  args
) {
  if (!(callingContext instanceof boundFunc))
    return sourceFunc.apply(context, args);
  var self = baseCreate(sourceFunc.prototype);
  var result = sourceFunc.apply(self, args);
  if (_.isObject(result)) return result;
  return self;
};

// Create a function bound to a given object (assigning `this`, and arguments,
// optionally). Delegates to **ECMAScript 5**'s native `Function.bind` if
// available.
_.bind = function(func, context) {
  if (nativeBind && func.bind === nativeBind)
    return nativeBind.apply(func, slice.call(arguments, 1));
  if (!_.isFunction(func))
    throw new TypeError("Bind must be called on a function");
  var args = slice.call(arguments, 2);
  var bound = function() {
    return executeBound(
      func,
      bound,
      context,
      this,
      args.concat(slice.call(arguments))
    );
  };
  return bound;
};
```

### js 实现千位分隔符

描述：数字的千位分隔符表示法，比如`126186312`转化为`126,186,312`。这个题目主要考察正则表达式，用正则可以很方便的进行字符串转化

```js
var result = "12345678".replace(/(?=\d{3}$)/g, ",");
console.log(result);
// => "12345,678"
```
