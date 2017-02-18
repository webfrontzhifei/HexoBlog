---
title: node.js命令行开发入门
date: 2016-11-23 16:35:25
tags:
- node
comments: true
description: 一直在gulp，grunt命令行中懵懵懂懂，向往已久，终于有了这篇命令行开发。也打开了之前的误区，yeoman 命令行开发是框架结构，而本文中的才是根本。开发命令行，完全可以不需要yeoman。另外本文中的博客命令行实战，也明朗了hexo博客的内核。后续会分析hexo博客源码。
categories:
- 技术
- 前端
- node
---
# 学习资源
* commander github地址：[commander](http://tj.github.io/commander.js/#Command.prototype.action)
* sass命令行实战 简书地址：[Node.js+commander开发命令行工具](http://www.jianshu.com/p/2cae952250d1)
* node自动构件化项目 慕课地址:[前端扫盲-之打造一个Node命令行工具](http://www.imooc.com/article/3156)

# 实战项目
[项目源码](https://github.com/webfrontzhifei/blogFrame)
命令行工具---开发一个静态博客系统(参考《Node.js实战》)
1. 编写命令行
项目初始化后,在根目录下新建bin子文件夹,在bin新建myblog文件，内容如下：
```javascript
#!/usr/bin/env node

var program = require('commander');

program.version(require('../package.json').version);

//help命令
program
	.command('help')
	.description('显示使用帮助')
	.action(function() {
		program.outputHelp();
	});

//create命令
program
	.command('create [dir]')
	.description('创建一个空的博客')
	.action(function(dir) {
		require('../lib/cmd_create.js')(dir);
	});

//preview命令
program
	.command('preview [dir]')
	.description('实时预览')
	.action(function(dir) {
		require('../lib/cmd_preview.js')(dir);
	});

//build命令
program
	.command('build [dir]')
	.description('生成整站静态html')
	.option('-o, --output <dir>', '生成的静态html存放目录')
	.action(function(dir, options) {
		require('../lib/cmd_build.js')(dir, options);
	});


program.parse(process.argv);

```
  其中第一行是核心，指定了node脚本运行命令行，其余为commander命令编写代码，参考commander API即可。
  然后，在package.json文件中指定bin，参考常用的gulp，express等命令。
  ```javascript
  {
  "name": "myblog",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "bin": {
    "myblog": "./bin/myblog.js"
  },
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "commander": "^2.9.0",
    "ejs": "^2.5.2",
    "express": "^4.14.0",
    "fs-extra": "^1.0.0",
    "markdown-it": "^8.1.0",
    "moment": "^2.17.0",
    "open": "^0.0.5",
    "rd": "^1.0.0",
    "serve-static": "^1.11.1",
    "swig": "^1.4.2"
  }
  }
  ```
  现在指定了myblog命令运行./bin/myblog文件了，要是命令行生效，需要安装到环境变量，可以通过两种方式。
  npm install . -g
  也就是安装本地模块包到全局环境变量。
  npm link
  也就是通过符号链接方式
  现在，就可以通过myblog help命令了
2. preview命令
  preview命令需要web服务器，使用express服务器即可。
  核心思路：
  * 通过express启动web服务，当监听到get请求时，通过路由读取相应的md文件，然后使用markdown-it进行转化为html文件，进行res.end(html)输出响应即可。
  * 文章元数据，也就是头部的title，date元数据，进行转化为post对象的属性，格式如下：
  ```javascript
  ---
  title:Node.js实战
  date:2015-05-15
  layout:post
  ---
  开始编写《node.js》实战
  ```
  * 增加模板使用ejs渲染模板文件，将渲染后的html通过res.end(html)文件输出。模板文件可以按标准的样式输出，并且添加样式文件style.css。
  * 渲染文章列表，也就是首页。通过rd模块遍历_posts目录下的所有文件，并按时间进行排序，最后通过layout模板下的index.html进行渲染首页。
3. build命令
  生成静态博客build命令,基本过程同上，只是将ejs根据模板渲染后的html，通过fs-extra模块进行写文件。同时通过open模块打开浏览器。
4. create命令
  创建新博客create命令，通过将必须的文件结构，放在tpl文件夹下，通过fs-extra模块的mkdir命令创建目录，通过moment模块处理时间，创建一个新博客的hello-world第一篇博客。
5. 第三方服务
  * 评论组件
    多说组件[多说](http://duoshuo.com/)
    Disqus[Disqus](https://disqus.com)
  * 分享组件
    加网[JiaThis](http://www.jiathis.com/)