# flexible
基于淘宝适配方案flexible 适配方案
## flexible解读及应用

原文: http://www.w3cplus.com/mobile/lib-flexible-for-html5-layout.html

大漠的文章（简洁）：https://github.com/amfe/article/issues/17

giuhub：https://github.com/amfe/lib-flexible

本文主要介绍的是如何使用Flexible库来完成H5页面的终端适配。为什么推荐使用Flexible库来做H5页面的终端设备适配呢？主要因为这个库在手淘已经使用了近一年，而且已达到了较为稳定的状态。
其实H5适配的方案有很多种，网上有关于这方面的教程也非常的多。移动端适配原理大同小异，大部分是通过控制根元素<html>的font-size值实现设备宽度的适配（通常是宽度）。

参考文章： <https://www.cnblogs.com/liangxuru/p/6970629.html>

<http://caibaojian.com/flexible-js.html>

<https://www.jianshu.com/p/6edffcd890e9>



[为腾讯微信优化的H5动效模板，帮助你快速构建全屏滚动型H5页面](<https://github.com/itxing666/wechat-h5-boilerplate>)

**方案一：浏览器缩放viewport**

>简单方便，像素失真

```javascript
	var phoneScale = parseInt(window.screen.width)/640;
	document.write('<meta name="viewport" content="width=640, initial-scale = '+phoneScale+', maximum-scale = '+phoneScale+', maximum-scale = '+phoneScale+', target-densitydpi=device-dpi">');
	}
```
**方案二：CSS MediaQuery**
>易理解，断点区间，安卓机型问题多，需特殊照顾

```css
	@media (min-width:340px) and (max-width: 359px) {
	    html,body{font-size: 66%;}
	}
	@media (min-width:360px) and (max-width: 374px) {
	    html,body{font-size: 70%;}
	}
	...
	
```
**方案三：淘宝flexible**

### 源码解读
**实际上：flexible计算dpr，通过width和dpr计算fontSize；对于dpr不等于1的设备，则通过viewport缩小，fontSize放大，达到设配高清屏的目的。**
flexible目前最新也是最稳定版本-0.3.2，实现很简单，主要做以下三件事：

>1.计算dpr，rem值
>2.动态改写meta-viewport标签
>3.动态改写html元素font-size的值

2、3点基于1的计算结果，那么问题就转化为计算dpr、rem，上源码：
**计算dpr核心代码**
```javascript
	var isIPhone = win.navigator.appVersion.match(/iphone/gi);
	var devicePixelRatio = win.devicePixelRatio;
	if (isIPhone) {
	    if (devicePixelRatio >= 3 && (!dpr || dpr >= 3)) {                
	        dpr = 3;
	    } else if (devicePixelRatio >= 2 && (!dpr || dpr >= 2)){
	        dpr = 2;
	    } else {
	        dpr = 1;
	    }
	} else {
	    // 其他设备下，仍旧使用1倍的方案
	    dpr = 1;
	}
```
>IOS下：dpr>=3 ==> 3,dpr>=2 ==> 2; 
>Android下：dpr === 1（有点无语）

**计算rem核心代码**
```javascript
	function refreshRem(){
	    var width = docEl.getBoundingClientRect().width;
	    if (width / dpr > 540) {
	        width = 540 * dpr;
	    }
	    var rem = width / 10;
	    docEl.style.fontSize = rem + 'px';
	    flexible.rem = win.rem = rem;
	}
```

**如果已有viewport或flexible标签则以此计算dpr（优先配置功能）**
```javascript
	var metaEl = doc.querySelector('meta[name="viewport"]');
    var flexibleEl = doc.querySelector('meta[name="flexible"]');
    if (metaEl) {
        //console.warn('将根据已有的meta标签来设置缩放比例');
        var match = metaEl.getAttribute('content').match(/initial\-scale=([\d\.]+)/);
        if (match) {
            scale = parseFloat(match[1]);
            dpr = parseInt(1 / scale);
        }
    } else if (flexibleEl) {
        var content = flexibleEl.getAttribute('content');
        if (content) {
            var initialDpr = content.match(/initial\-dpr=([\d\.]+)/);
            var maximumDpr = content.match(/maximum\-dpr=([\d\.]+)/);
            if (initialDpr) {
                dpr = parseFloat(initialDpr[1]);
                scale = parseFloat((1 / dpr).toFixed(2));
            }
            if (maximumDpr) {
                dpr = parseFloat(maximumDpr[1]);
                scale = parseFloat((1 / dpr).toFixed(2));
            }
        }
    }
```

**动态改写meta-viewport、html的data-dpr属性（默认配置）**
```javascript
	docEl.setAttribute('data-dpr', dpr);
	if (!metaEl) {
	    metaEl = doc.createElement('meta');
	    metaEl.setAttribute('name', 'viewport');
	    metaEl.setAttribute('content', 'initial-scale=' + scale + ', maximum-scale=' + scale + ', minimum-scale=' + scale + ', user-scalable=no');
	    if (docEl.firstElementChild) {
	        docEl.firstElementChild.appendChild(metaEl);
	    } else {
	        var wrap = doc.createElement('div');
	        wrap.appendChild(metaEl);
	        doc.write(wrap.innerHTML);
	    }
	}
```

### viewport知识补充

如果你对视窗viewport不是很了解，建议读读[《消除viewport的疑惑-移动网页开发》](https://www.zybuluo.com/gongzhen/note/170557)，写的非常好！
通常的，移动端页面会写上这么一句：
```html
	<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1, maximum-scale=1, user-scalable=no" />
```
flexible的viewport是这样写的：
```javascript
	metaEl.setAttribute('content', 'initial-scale=' + scale + ', maximum-scale=' + scale + ', minimum-scale=' + scale + ', user-scalable=no');
```
读到这里会发现，flexible没有直接定义device-width？
```html
	<meta name="viewport" content="initial-scale=1">
	<meta name="viewport" content="width=device-width">
```

> 这两句代码能达到一样的效果，也可以把当前的的viewport变为 ideal viewport。缩放是相对于什么来缩放的？因为这里的缩放值是1，也就是没缩放，但却达到了 ideal viewport 的效果，所以，那答案就只有一个了，缩放是相对于 ideal viewport来进行缩放的，当对ideal viewport进行100%的缩放，也就是缩放值为1的时候，不就得到了 ideal viewport吗？事实证明，的确是这样的。当两者同时存在时取其最大值。

**附：**
```javascript
	device-width      ==> win.screen.width                      始终是竖屏结果
	ideal-viewport    ==> docEle.getBoundingClientRect().width  横竖屏结果不一样
	initial-scale = 1 ==> 100% * ideal-viewport
```
<hr>

### 使用
在head引入flexible.js

> 使用flexible就不要设置metaviewport了，因为会阻断后面的执行，详情看源码；除非你想自定义dpr

```html
	//直接加载cdn文件（其中css文件非必须）
	<script src="http://g.tbcdn.cn/mtb/lib-flexible/0.3.4/??flexible_css.js,flexible.js"></script>
	//或下载到项目目录下
	<script src="/local-assets/js/flexible/flexible-0.3.2.js"></script>
```
其他的事都是css的事，根据你的PSD尺寸，如640px宽，那么rem基准就是640/10=64px，有一个64px高的div就是：height:1rem;
推荐 [sublimeText3 cssrem插件](https://github.com/flashlizi/cssrem) 自动完成计算