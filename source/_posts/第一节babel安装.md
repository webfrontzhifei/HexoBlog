---
title: 第一节babel安装
date: 2016-11-12 17:40:44
tags:
- es6
comments: true
categories:
- 技术
- 前端
---
# e6学习
## 第一节babel安装
参考地址  
[babel官网]( http://babeljs.io/)  
[郭永峰博客](http://guoyongfeng.github.io/idoc/html/React%E8%AF%BE%E7%A8%8B%E4%B8%93%E9%A2%98/Babel%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97.html#t14babel-runtime)

1. 安装babel命令行  
   
    全局安装
	
		npm install babel-cli -g  

	本地安装  

		npm install --save-dev babel-cli
  
	然后进行设置在package.json中添加 
	 
		{
			"scripts: {  
				"build": "babel src -d lib"
			}
		}
	  
	然后运行`npm run build`命令即可。  
   
	简单解释下命令行：    

		babel lesson1.js --out-file lesson1-compiled.js  

	表示输出转译lesson1.js到lesson1-compiled.js文件  
	
		babel src --out-dir lib  
	
	表示转译src文件夹下的所有文件到lib文件夹下  
2. 编译插件  
	使用步骤一中的命令实际为原样输出，这是因为没有安装相应的插件。官网提示安装最新的插件`babel-preset-latest`. 
	
		npm install --save-dev babel-preset-latest  
	
    然后添加到`.babelrc`配置文件中

		{
			"presets": [
				"latest"
			]
		}
	 
 babel默认只转化新的JavaScript语法（syntax），而不转化新的API，比如Iterator，Generator，Set，Maps，Proxy，Reflect，Symbol，Promise等，详情查看[definitions.js](https://github.com/babel/babel/blob/master/packages/babel-plugin-transform-runtime/src/definitions.js).  
为了使用最新的API，安装腻子polyfill插件  

		npm instaall --save-dev babel-polyfill
	 
	使用的文件顶部引入

		import "babel-polyfill";
	