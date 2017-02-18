---
title: es6学习环境搭建
date: 2016-11-12 17:40:44
tags:
- es6
comments: true
description: 系属es6学习系列文章，本章节主要是es6学习环境的搭建，通过gulp watch监控文件变化，然后自动运行babel任务，nodemon重新运行文件。
categories:
- 技术
- 前端
---
### e6学习系列
#### 第二节 学习es6环境搭建
1.	gulp 任务配置检测文件变化
	gulpfile.js配置
	```javascript
	var gulp = require('gulp'),
	gulpLoadPlugins = require('gulp-load-plugins');

	var plugins = gulpLoadPlugins();

	gulp.task('babel', function() {
		return gulp.src("src/**/*.js")
			.pipe(plugins.babel())
			.pipe(gulp.dest("lib"));
	});
	
	gulp.task('watch', ['babel'], function() {
		gulp.watch('src/**/*.js', ['babel']);
	});
	```
2.	nodemon自动运行。
	nodemon.json配置
	```javascript
		 {
		  "restartable": "rs",
		  "ignore": [
		    ".git",
		    "node_modules/**/node_modules"
		  ],
		  "verbose": true,
		  "execMap": {
		    "js": "node --harmony"
		  },
		  "events": {
		    "restart": "osascript -e 'display notification \"App restarted due to:\n'$FILENAME'\" with title \"nodemon\"'"
		  },
		  "watch": [
		    "lib/**/*.js"
		  ],
		  "env": {
		    "NODE_ENV": "development"
		  },
		  "ext": "js json"
		}
	```
3. 运行代码，只需要`nodemon lib/lesson1.js`

4. node.js对es6的支持情况93%，详情参考[e6支持](https://kangax.github.io/compat-table/es6/) .