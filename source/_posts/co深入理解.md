---
title: co深入理解
date: 2016-11-29 21:05:37
tags:
- node.js
- 异步编程系列
- co.js
description: 异步编程系列文章，学习koa必经之路。co源码分析！！！
comments: true
categories:
- 技术
- 前端
---
### co介绍
  个人理解：解决Generator自动运行问题，使代码更加优雅。原文：Generator based control flow goodness for nodejs and the browser, using promises, letting you write non-blocking code in a nice-ish way.
### co API
  原文：
  co@4.0.0 has been released, which now relies on promises. It is a stepping stone towards the async/await proposal. The primary API change is how co() is invoked. Before, co returned a "thunk", which you then called with a callback and optional arguments. Now, co() returns a promise.
  co从4.0.0版本后，开始依赖promise。这是为了以后更方便向es7的async/await转移。API主要发生的变化时，以前co()调用后返回的是一个thunk函数（只有一个参数，而且这个参数只能是callback），现在返回的是promise对象。
  
 co(fn*).then(val => )，Returns a promise that resolves a generator, generator function, or any function that returns a generator.
  ```javascript
  co(function* (){
    return yield Promise.resolve(true);
  }).then(function (val) {
    console.log(val);
  }, function(err) {
    console.error(err.stack);
  });
  ```
### co深入原理
  例如，Generator函数，读取两个文件。
  ```javascript
  var gen = function* () {
    var f1 = yield readFile('atom.html');
    console.log(f1.toString());
    var f2 = yield readFile('package.json');
    console.log(f2.toString());
  }
  ```
  使用co函数库，可是不用编写Generator函数的执行器
  ```javascript
  var co = require('co');
  co(gen).then(function() {
    console.log('Generator 自动执行');
  });
  ```
  co函数库的原理就是将两个自动执行器(thunk函数和Promise对象),包装成一个库，使用co的前提，Generator函数的yield命令后面只能是（Thunk函数或者Promise对象）。
  
  下面介绍promise自动执行器，将上例中的readFile包装成一个Promise对象。
  ```javascript
  var fs = require('fs');
  
  var readFile = function(path) {
    return new Promise(function(resolve, reject) {
      fs.readFile(path, function(err, data) {
        if(err){
          reject(err);
        }else {
          resolve(data);
        }
      });
    });
  };
  
  var gen = function* () {
    var f1 = yield readFile('atom.html');
    console.log(f1.toString());
    var f2 = yield readFile('package.json');
    console.log(f2.toString());
  }
  ```
  
  如果，手动自行上述Generator函数。
  ```javascript
  var g = gen();
  g.next().value.then(function(f1){
    g.next(f1).value.then(function(f2) {
      g.next(f2);
    });
  });
  ```
  将上述过程用一个函数run自动执行，也就是
  ```javascript
  function run(gen) {
    var g = gen();
    function next(data) {
      var result = g.next(data);
      if(result.done)
        return result.value;
      result.value.then(function(data){
        next(data);
      });
    }
    next();
  }
  
  run(gen);
  ```
  上述也就是Generator函数的自动执行过程。而co库就是做了这样一件事情。
  
  ### co源码分析
  首先，co函数参数为Generator函数，返回一个Promise对象。
  
  ```javascript
  function co(gen) {
    var ctx = this;
    
    return new Promise(function(resolve, reject) {
    });
  }
  ```
  
  在返回的Promise对象里，co先检查参数gen是否为Generator函数。如果是，就执行，生个generator生成器对象，否则返回，修改状态为resolved。
  ```javascript
  function co(gen) {
    var ctx = this;
    return new Promise(function(resolve, reject) {
      if(typeof gen === 'function'){
        gen = gen.call(ctx);
      }
      if(!gen || typeof gen.next != 'function') {
        return resolve(gen);
      }
    }
  }
  ```
  
  接着，co将Generator函数的自动执行方法（上文中的run）中的next方法，包装成为了onFulfilled（）方法，主要是为了捕捉异常抛出的错误。而next()逻辑同上。
  ```javascript
  function co(gen) {
    var ctx = this;
    return new Promise(function(resolve, reject) {
      if(typeof gen === 'function'){
        gen = gen.call(ctx);
      }
      if(!gen || typeof gen.next != 'function') {
        return resolve(gen);
      }
      onFulfilled();
      
      function onFulfilled(res) {
        var ret;
        try {
          ret = gen.next(res);
        }catch(e) {
          return reject(e);
        }
        next(ret);
        return null;
      }
      
      function onRejected(err) {
        var ret;
        try {
          ret = gen.throw(err);
        } catch(e) {
          return reject(e);
        }
        next(ret);
      }
      
      function next(ret) {
        if(ret.done) return resolve(ret.value);
        var value = toPromise.call(ctx, ret.value)
        if(value && isPromise(value)) return value.then(onFulfilled, onRejected);
        return onRejected(new TypeError('You may only yield a function, promise, generator, array, or object, '+ 'but the following object was passed: "' + String(ret.value) + '"'));
      }
    });
  }
  ```
  核心代码也就是上述了，其他都是围绕toPromise方法了。
  
  ```javascript
  /**
   * Convert a `yield`ed value into a promise.
   *
   * @param {Mixed} obj
   * @return {Promise}
   * @api private
   */
  
  function toPromise(obj) {
    if (!obj) return obj;
    if (isPromise(obj)) return obj;
    if (isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
    if ('function' == typeof obj) return thunkToPromise.call(this, obj);
    if (Array.isArray(obj)) return arrayToPromise.call(this, obj);
    if (isObject(obj)) return objectToPromise.call(this, obj);
    return obj;
  }
  ```
  
  首先看isPromise(obj)源码进行分析,直接判断了then属性。
  ```javascript
  function isPromise(obj) {
    return 'function' == typeof obj.then;
  }
  ```
  然后，isGeneratorFunction(obj)判断是否为generator生成器函数。
  ```javascript
  function isGeneratorFunction(obj) {
    var constructor = obj.constructor;
    if (!constructor) return false;
    if ('GeneratorFunction' === constructor.name || 'GeneratorFunction' === constructor.displayName) return true;
    return isGenerator(constructor.prototype);
  }
  function isGenerator(obj) {
    return 'function' == typeof obj.next && 'function' == typeof obj.throw;
  }
  ```
  上述理解很简单，isGeneratorFunction()首先判断obj的constructor的name属性，如果是'GeneratorFunction'，就返回true。另外，如果，该函数的原型是一个generator对象，那么这个构造函数也是。
  对于toPromise的第二行代码
  ```javascript
  if(isGeneratorFunction(obj) || isGenerator(obj)) return co.call(this, obj);
  ```
  将会产生迭代的效果，也就是类似于yield*的效果。
  demo:
  ```javascript
  var gen = function * () {
  	var f1 = yield readFile('package.json');
  	console.log(f1);
  	var f2 = yield readFile('scheme.js');
  	console.log(f2);
  };
  
  var gen2 = function *() {
  	var f3 = yield gen;
  	var f4 = yield readFile('app.js');
  	console.log(f4);
  }
  // var g = gen();
  co(gen2).then(function() {
  	console.log('执行完成');
  });
  ```
  
  toPromise的第三行代码,判断并转化thunk函数
  ```javascript
  if ('function' == typeof obj) return thunkToPromise.call(this, obj);
  ```
  thunkPromise源码如下：
  ```javascript
  function thunkToPromise(fn) {
    var ctx = this;
    return new Promise(function (resolve, reject) {
      fn.call(ctx, function (err, res) {
        if (err) return reject(err);
        if (arguments.length > 2) res = slice.call(arguments, 1);
        resolve(res);
      });
    });
  }
  ```
  由上述源码，很明显看到thunkToPromise(fn)就是new一个promise对象，并在fn的callback中将调用resolve(data)方法。
  
  toPromise的第四行，数组判断。
  ```javascript
  if(Array.isArray(obj)) return arrayToPromise.call(this, obj);
  ```
  arrayToPromise的源码
  ```javascript
  function arrayToPromise(obj) {
    return Promise.all(obj.map(toPromise, this));
  }
  ```
  从上述源码Promise.all方法，可以看出，如果yield后为数组，将会并行处理异步操作。
  
  toPromise的第5行，转化的为普通Object。
  ```javascript
  if(isObject(obj)) return objectToPromise.call(this, obj);
  ```
  objectToPromise源码如下：
  ```javascript
  function objectToPromise(obj) {
    var results = new obj.constructor();
    var keys = Object.keys();
    var promises = [];
    for(var i = 0; i < keys.length; i++) {
      var key = keys[i];
      var promise = toPromise.call(this, obj[key]);
      if(promise && isPromise(promise)) defer(promise, key);
      else results[key] = obj[key];
    }
    return Promise.all(promises).then(function() {
      return results;
    }
    
     function defer(promise, key) {
        results[key] = undefined;
        promises.push(promise.then(function(res) {
          results[key] = res;
        }
      }
  }
  ```
  上述代码实质上也是通过Promise.all去并行实现调用object所有属性值转化的promise对象。但是它使用了一个defer方法也就是，在defer中定义了每个promise的then方法将结果放入到results中。可以看到在Promise.all的then回调总直接返回了results。
 
 至此，核心API就结束了，还有一个wrap方法。
 源码：
 ```javascript
 co.wrap = function (fn) {
   createPromise.__generatorFunction__ = fn;
   return createPromise;
   function createPromise() {
     return co.call(this, fn.apply(this, arguments));
   }
 };
 ```
 原文这样解释：Convert a generator into a regular function that returns a Promise.
 也就是将一个generator转化为一个产生promise的普通函数createPromise，co.call(this,fn.apply(this, arguments)),生成promise对象。同时为了绑定fn与createPromise函数之间的联系，添加了__generatorFunction__属性（没有查到实质的作用）。
 
### 总结
 上文就是co模块的全部内容了，后续会陆续将异步编程系列的generator，promise等完成。
### 参考资源
  [co模块github](https://github.com/tj/co)
  [co函数的含义与用法---阮一峰](http://www.ruanyifeng.com/blog/2015/05/co.html)
  [co源码解读](http://www.cnblogs.com/jiasm/p/5800210.html)
  [异步编程之co---源码分析](http://www.html-js.com/article/3016)
  [koa源码分析系列（二）co的实现](http://purplebamboo.github.io/2014/05/24/koa-source-analytics-2/)
  [Node.js实战---第二季]()
  