---
title: highcharts经验之谈一
date: 2016-09-13 18:37:56
tags:
- highcharts
comments: true
categories:
- 技术
- 前端
---
* 怎么改变highart type为折线图line时，设置legend symbol的方法。

    两种方式:

    [google方案](http://stackoverflow.com/questions/27510810/highcharts-make-the-legend-symbol-a-square-or-rectangle)

```javascript
Highcharts.seriesTypes.line.prototype.drawLegendSymbol =
     Highcharts.seriesTypes.area.prototype.drawLegendSymbol;

chart: {

      events: {
          load: function () {
              $(".highcharts-legend-item path").attr('stroke-width', 16);
          },
          redraw: function () {
              $(".highcharts-legend-item path").attr('stroke-width', 16);
          }
      }
  },
```
* 怎么对齐legend文字与symbol

  1.在面积图areachart下实现对齐,默认情况下当改变legend的text文本时，symbol就会与text发生位置移，此时首先想到的方案自然是改变legend的itemStyle，

````javascript
itemStyle: CSSObject
CSS styles for each legend item. Only a subset of CSS is supported, notably those options related to text.
Defaults to { "color": "#333333", "cursor": "pointer", "fontSize": "12px", "fontWeight": "bold" }.
````
可以看到itemStyle并没有关于css的position相关属性，不甘心？？试试呗！是可以的，可以的么？神奇的是，itemStyle的marginTop没有作用，但是hover上去后，鼠标再移开后， 奇迹才发生，对齐了，marginTop产生了效果，然并卵，并不是我们想要的。

经试验，此时使用useHTML：true，改变text的svg渲染方式，使用html标签渲染，可以实现对齐。

完整代码实现，
````javascript
   legend: {
        layout: 'horizontal',
        align: 'center',
        symbolRadius: 0,
        symbolHeight: 16,
        symbolWidth: 16,

       itemStyle: {
            color: '#66686C',
            fontSize: '14px',
            lineHeight: '14px',
            fontWeight: 'normal',
            fontFamily: 'microsoft yahei'
        },
        itemHoverStyle: {
            color: '#1E2330',
            fontSize: '14px'
        },
        useHTML: true,
        verticalAlign: 'bottom',
        borderWidth: 0
   },

````
  2.在linechart下实现symbol与text对齐，上文中提到了linechart下实现symbol为正方形的实现方案。由于默认linechart legend的symbol不存在symbol为方形的情况，实际上linechart下会发现，当text的font-size14px时，symbol与text明显是底部对齐，与我们的设计湿们的效果不一样啊。

  此处到了转折点：

场景1，默认symbol，产品接受，设计师接受，使用areachart下的对齐方案即可，useHTML的方式；

场景2，设计师必须要方形square，先马上几行上文提到的代码，实现方形呗，useHTML：false,没有错，一定是false，垂直居中对齐了。。

场景3, 场景2埋下了当时炸弹啊，坑有些深啊，通过我们的设计，切换tab时，发现在第一个tab下areachart与第三个tab下linechart时，轻微抖动，拿工具量量呗，X轴有2~3px的偏移，反复试验，跟chart的type无关，而是useHTML：true，与useHTML：false两种情况下发生的，这可怎么办？？首先，为了不抖动，果断改变useHTML：true，3个tab下的chart必须统一啊！再去检查代码

`````javascript
<g class="highcharts-legend" zIndex="7" transform="translate(272,275)"><g zIndex="1"><g><g class="highcharts-legend-item" zIndex="1" transform="translate(8,3)"><rect x="0" y="12" width="16" height="16" zIndex="3" fill="#00cc26"></rect></g><g class="highcharts-legend-item" zIndex="1" transform="translate(197,3)"><rect x="0" y="12" width="16" height="16" zIndex="3" fill="#0067ed"></rect></g><g class="highcharts-legend-item" zIndex="1" transform="translate(386,3)"><rect x="0" y="12" width="16" height="16" zIndex="3" fill="#002982"></rect></g></g></g></g>
`````
发现useHTML：true后，legend的symbol与text是分离的，这里g.highcharts-legend只控制legend symbol的位置了，而且是通过transform的translate控制位置的，那就easy了，直接仿照上文中实现方形的两个事件load和redraw中，改变tramslateY属性就可以了嘛！

````javascript
/*
* 这段逻辑是为了处理legend symbol与text不对齐问题，hack方法，改变tranlateY，上移3px
*/
var $legend = $(host).find('.highcharts-legend').eq(0);
var translateInit = $legend.attr('transform');
if(translateInit) {
	var translateY = parseInt(translateInit.slice(translateInit.indexOf(',')+1, -1));
	translateInit = translateInit.slice(0, translateInit.indexOf(',')+1) + (translateY-3) + ')';
}
$legend.attr('transform', translateInit);

`````
革命尚未成功，同志仍需努力。是真的，是真的，坑又出现了，上面的逻辑，是在load与redrew事件中的，redrew，redrew，问题就出现在这里，redrew不仅在window resize情况下，highchart是支持点击legend后，当前legend多代表的series data（数据列）隐藏掉的，再次单击，重现出现（redrew事件触发了），会发现，使用，symbol跑上去了，多次点击，越跑越远，方案解决中。。。
已解决
添加标示flag，
完整代码
`````javascript

var legendClicked = false;  //标示位
var legendClickHandler = function (event) {

	var visibleSeries = 0;
	for (var i = 0; i < this.chart.series.length; i++) {
		if (this.chart.series[i].visible) {
			visibleSeries++;
		}
	}
	if (visibleSeries === 1 && event.target.visible) {
		return false;
	}
	legendClicked = true;
};
chart: {
	events: {
		load: function () {

			/*
			* 这段逻辑是为了处理legend symbol与text不对齐问题，hack方法，改变tranlateY，上移3px
			*/
			var $legend = $(host).find('.highcharts-legend').eq(0);
			var translateInit = $legend.attr('transform');
			if(translateInit) {
				var translateY = parseInt(translateInit.slice(translateInit.indexOf(',')+1, -1));
				translateInit = translateInit.slice(0, translateInit.indexOf(',')+1) + (translateY-3) + ')';
			}
			$legend.attr('transform', translateInit);

			$('.highcharts-legend-item path').attr('stroke-width', 16);
			$('.highcharts-legend-item path').attr('stroke-height', 16);
		},
		redraw: function () {

			/*
			* 这段逻辑是为了处理legend symbol与text不对齐问题，hack方法，改变tranlateY，上移3px
			*/
			if(!legendClicked) {
				var $legend = $(host).find('.highcharts-legend').eq(0);
				var translateInit = $legend.attr('transform');
				if(translateInit) {
					var translateY = parseInt(translateInit.slice(translateInit.indexOf(',')+1, -1));
					translateInit = translateInit.slice(0, translateInit.indexOf(',')+1) + (translateY-3) + ')';
				}
					$legend.attr('transform', translateInit);
				}
				legendClicked = false;

				$('.highcharts-legend-item path').attr('stroke-width', 16);
				$('.highcharts-legend-item path').attr('stroke-height', 16);
			}
		}
	}


options.plotOptions = {
		series: {
		    events: {
				legendItemClick: legendClickHandler
			}
		}
	};
`````

* 怎么设置x轴坐标点的间隔为4

````javascript

xAxis: {
    labels: {
        step: 4
    }
}
````

* total字段在area stack中存在，在line中处理方式

``` javascript
//在area stack为default下，
this.points[0].total;//total指的是stack值相同的累积和
//在line chart
$.each(this.points, function () {
    total += this.y;
});

````
* tooltip的坑好深
  tooltip是highcharts里高大上的玩意，由于api中的formater函数，强大的不得了。

```javascript
    Callback function to format the text of the tooltip. Return false to disable tooltip for a specific point on series.

A subset of HTML is supported. The HTML of the tooltip is parsed and converted to SVG, therefore this isn't a complete HTML renderer. The following tabs are supported: <b>, <strong>, <i>, <em>, <br/>, <span>. Spans can be styled with a style attribute, but only text-related CSS that is shared with SVG is handled.

````
  坑在哪里呢，formatter实现，默认效果，tooltip每行前是对应legend symbol的，使用了formatter，只能html实现。
  ```javascript
  sbody += '<span style="background-color: ' + this.series.color + ';margin-right: 8px; width: 10px;height: 10px;display: inline-block;">' + '' + '</span>'
  ````
为什么还说是坑呢？开始使用的方案（google下搜索的），直接使用特殊字符（word下的特殊字符），方形的，谁让电脑分辨率差啊，设计师同学说，你那分明是长方形了，只能ctrl+，放大浏览器，果然，果然。。。

tooltip这么顺利么？NO！怎么可能！视觉走查，你的tooltip怎么消失的有延迟啊，有延迟。好吧，可是哪里有问题呢，N久，没有解决，最后听了另一个小组的同学的方案，把tooltip的backgroundColor属性改为transparent，在formater里设置background，好了，好了。据说，是他的导师的建议，果然是老司机。 致敬老司机。

tooltip坑，继续！tooltip在点很多的情况下，性能如此之差，插件tooltip delay before display，无非就是处理频发触发。。

* 设计师的slash，在面积图中使用斜线
查遍了api，没有！还好，在highcharts的官网下提供了插件（隐藏的很深，在head头的最右侧plugins，pattern-fill插件，改变svg绘制，实现。
源码中核心
````javascript
Highcharts.wrap(Highcharts.SVGElement.prototype, 'fillSetter', function (proceed, color, prop, elem) {
	var markup,
		id,
		pattern,
		image;
	if (color && color.pattern && prop === 'fill') {
		id = 'highcharts-pattern-' + idCounter++;
		pattern = this.renderer.createElement('pattern')
			.attr({
				id: id,
				patternUnits: 'userSpaceOnUse',
				width: color.width,
				height: color.height
			})
			.add(this.renderer.defs);
		image = this.renderer.image(
			color.pattern, 0, 0, color.width, color.height
		).add(pattern);
		elem.setAttribute(prop, 'url(' + this.renderer.url + '#' + id + ')');
	} else {
		return proceed.call(this, color, prop, elem);
	}
});
````
具体实现
````javascript
//填充斜线
data[m].fillColor = {
	width:6,
	height: 6,
	pattern: '/static_proxy/ad/src/themes/qidian/phone/images/stripe.png'
};
//color斜线，legend Symbol便能以斜线显示，但是注意此时不能显示豆点
data[m].color = {
	width:6,
	height: 6,
	pattern: '/static_proxy/ad/src/themes/qidian/phone/images/stripe.png',\
	parentColor: '#CCF5D4'  //透明度后的颜色
};
marker: {
	enabled: false,
},
//tooltip斜线显示
tooltip: {
	formatter: function () {
		var s = '<div style="margin-bottom: 5px">' + this.x + '</div>';

		$.each(this.points, function () {
			if(this.series.color && this.series.color.pattern) {
				s += '<span style="background-image: url(' + this.series.color.pattern + ');background-color:'     +this.series.color.parentColor + ';margin-right: 8px; width: 10px;height: 10px;display: inline-block;">' + '' + '</span>'+ this.series.name + ': ' + this.y + '<br />';
			}else {
				s += '<span style="background-color: ' + this.series.color + ';margin-right: 8px; width: 10px;height: 10px;display: inline-block;">' + '' + '</span>'+ this.series.name + ': ' + this.y + '<br />';
			}
		});

		return '<div style="background:#1E2330;padding:10px;border-radius:2px;">' + s + '</div>';
	}
},
````
* linechart下的平均线

场景：linechart，3条刻度线gridline即可，0%，avg%（平均值），100%；

    方案一：首选plotline方案，也是通常的考虑方式，弊端（不宜控制label的x，y位置）

    方案二: 直接tickPosition控制，因为场景特殊，三个tickpostion很容易实现，完美！
* loading问题

直接代码，
```javascript
loading: {
   labelStyle: {
        backgroundImage: 'url("/static_proxy/ad/src/themes/qidian/phone/images/loading2.gif")',
        position: 'absolute',
        display: 'block',
        width: '30px',
        height: '30px',
	    marginLeft: '-15px',
        left: '50%'
   },
	style: {
        backgroundColor: 'white',
        opacity: 0.8
	}
}
```
loading相关方法，this.chart.showLoading();this.chart.hideLoading();
* 优先级问题

科普一下，查阅highcharts api发现，很多配置项是重复的，所以相同的配置会出现优先级问题，highcharts的优先级是，series > plotOptions.* > plotOptions.series

在实际应用中，尤其是一个容器中包含多个chart的情况下，合理利用优先级，可以大大减少代码量。针对图表的常用配置，通过，plotOptions.series设置，对于所有相同类型的图表的设置，建议通过plotOptions.*设置，针对特定一个图表的设置，通过series设置。组件化开发，一定注意优先级问题，将通用config放在plotOptions.*设置。