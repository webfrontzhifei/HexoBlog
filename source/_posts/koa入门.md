---
title: koa入门
date: 2016-11-29 21:21:50
tags:
- node.js
- web框架
- koa系列
comments: true
description: 一直在用Express，对koa一直不曾触碰，忍不住要入门了，作为紧跟前端技术的菜鸟，飞...
categories:
- 技术
- 前端
---
### 第一个app
  ```javascript
  'use strict';
  
  let koa = require('koa');
  let app = koa();
  
  app.use(function*(next){
  	console.log('1');
  	yield next;
  	console.log('5');
  });
  
  app.use(function*(next) {
  	console.log('2');
  	yield next;
  	console.log('4');
  });
  
  app.use(function*(next) {
  	console.log('3');
  	this.body = 'hello world';
  });
  
  app.listen(4000);
  
  ```
  上述代码运行结果为1，2，3，4，5。显而易见，运行顺序为第一个中间件yield之前，然后运行第二个中间件yield之前，然后运行第三个中间件yield之前。
### koa中间件机制
  中间件的实质运行是什么呢？推理，能产生上述顺序的只有一种generator function机制。我们知道co库可以yieldable的对象是generator function，promise，array等。当为generator function时，就会是如下的顺序。
 例如： 
 
 ```javascript
  var gen1 = function * () {
   console.log('1');
   yield gen2;
   console.log('5');
  }
  var gen2 = function* () {
    console.log('2');
    yield gen3;
    console.log('4');
  }
  var gen3 = function* () {
    console.log('3');
  }
  co(function * () {
    yield gen1;
  });
 ```
 为了容易理解，就是
 ```javascript
  var gen1 = function * () {
    console.log('1');
    yield function* () {
               console.log('2');
               yield function* () {
                          console.log('3');
                        };
               console.log('4');
             };
    console.log('5');
   }
   co(function * () {
     yield gen1;
   });
 ```
 将上述代码中的每个gen提取为next参数，那么就是
 ```javascript
 var gen1 = function * (next) {
    console.log('1');
    yield next;
    console.log('5');
   }
   var gen2 = function* (next) {
     console.log('2');
     yield next;
     console.log('4');
   }
   var gen3 = function* () {
     console.log('3');
   }
 ```
 因此，koa中间件的机制，无非就是将后面一个中间件generatorFunction作为next参数穿给前一个中间件。
 也就是gen1的next参数为gen2，gen2的next参数为gen3.
 ### 简易SimpleKoa
 ```javascript
 var co = require('co');
 
 function SimpleKoa(){
  this.middlewares = [];
 }
 SimpleKoa.prototype.use = function(mw) {
  this.middlewares.push(mw);
 }
 SimpleKoa.prototype.listen = function() {
  this._run();
 }
 SimpleKoa.prototype._run = function() {
  var ctx = this;
  var middlewares = ctx.middlewares;
  return co(function *() {
    var prev = null;
    var i = middlewares.length;
    
    //依次遍历，将next参数传入。
    while(i--) {
      prev = middlewares[i].call(ctx, prev);
    }
    yield prev;
  });
 }
 var app = new simpleKoa();
 app.listen(3000);
 ```
 
### 自定义koa中间件timer
  ```javascript
  // middleware/timer.js
  module.exports = function() {
    return function *(next) {
      const path = this.path;
      const start = new Date;
      yield next;
      const end = new Date;
      console.log(`${path} response time: ${ end - start }ms`);
    }
  };
  //app.js
  const timer = require('./middleware/timer.js');
  app.use(timer());
  ```
  如果为timer中间件加上option配置参数，那么就可以改为如下
  ```javascript
  module.exports = function(options) {
  	return function* (next) {
  		const path = this.path;
  		const start = new Date;
  		const method = this.request.method;
  		if(method != options.filter.method)
  			start = 0;
  		yield next;
  		const end = new Date;
  		if(start !== 0 && end-start > options.filter.min) {
  			console.log(options.format.replace(/:url/g, path).replace(/:time/g,`${end-start}ms`));
  		}
  	}
  }
  //app.js
  app.use(timer({
      format: ':url :time',
      filter: {
          min: 0,
          method: 'GET'
      }
  }));
  ```
### 参考文献
  [koa文档](http://koa.bootcss.com/)
  [gitbook1](http://book.apebook.org/minghe/koa-action/start/router.html)
  [半小时掌握koa](http://www.jianshu.com/p/07008bc834c5)
  [koa技术分享](http://www.jianshu.com/p/225ff3e8fa18)
  [koa-router](https://github.com/alexmingoia/koa-router)
### koa examples分析[Examples](https://github.com/koajs/examples)
  1. 404
  ```javascript
  var koa = require('koa');
  
  var app = module.exports = koa();
  
  app.use(function *pageNotFound(next) {
    yield next;
  
    if (404 != this.status) return;
  
    // we need to explicitly set 404 here
    // so that koa doesn't assign 200 on body=
    this.status = 404;
  
    switch (this.accepts('html', 'json')) {
      case 'html':
        this.type = 'html';
        this.body = '<p>Page Not Found</p>';
        break;
      case 'json':
        this.body = {
          message: 'Page Not Found'
        };
        break;
      default:
        this.type = 'text';
        this.body = 'Page Not Found';
    }
  });
  
  if (!module.parent) app.listen(3000);
  ```
    &nbsp;&nbsp; &nbsp;&nbsp;代码中两处地方值得思考，首先判断this.status != 404,直接return,这说明上文中已经有了this.body的响应。注意，判断this.status == 404后，又显式进行this.status = 404赋值的原因是，在下文switch语句块中对this.body进行了赋值，在koa的内部机制中，当this.body有值时，this.status会默认赋值为200，除非显式的进行明确对this.status赋值。
    &nbsp;&nbsp; &nbsp;&nbsp;另外，最后一句代码module.parent是require该模块的模块，也就是只有该模块没有被require调用，而是直接调用时，才进行app.listen(3000);
      