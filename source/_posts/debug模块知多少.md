---
title: debug模块知多少
date: 2016-12-06 15:41:17
tags:
- node.js
comments: true
description: 忽然想用debug了，却忘记了怎么用，记录一下吧...
categories:
- 技术
- 前端
---
### node自带debug
  有详细的文档，就不多说了。
  [Debugger模块](http://nodejs.cn/doc/node/debugger.html)
### debug 模块
  debug模块主要用于log打印信息（个人感觉作用不大）。
  唯一注意的地方是：windows环境下命令行，set DEBUG=mydebug:*  &  node app.js
  [debug](https://github.com/visionmedia/debug)
### webstorm调试
  直接使用webstorm打断点，最实用！！！
### 参考资料
  [开始学nodejs——调试篇](http://www.tuicool.com/articles/Fzyaa2)
