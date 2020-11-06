---
layout: post
title: 现代浏览器的渲染过程与优化思路
date:   2017-7-17 12:30:00 +0800
category: brower 
tag: [brower]
---

* content
{:toc}
 
# 前言

- 在`youtude`上学习了[Browser Rendering Optimization][1]的课程，做学习笔记整理
- 本篇讲述的是, 单个请求一直到将像素填充到屏幕上的简单流程
- 本文的图片和内容均来自课程内容——[Browser Rendering Optimization][1]


# 概括渲染的流程

- prase HTML 解析富文本
- recalculate style (DOM + CSS = render tree) 将DOM和CSS合成实际显示的树状节点
- multi surfaces, layouts and composite 合成图层
- raster, draw bitmap 矢量转位图，光栅器生成像素点

# 详细流程

## prase HTML

请求页面

![brower-render-prase-html](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-prase-html.png)

- 浏览器向服务端发出获取请求，如 `GET / HTTP/1.1`
- 服务端返回HTML，浏览器会采取非常机智的措施并提前解析，生成DOM(document object model)
- 下一步就是将HTML与CSS结合起来

## recalculate style, 

![brower-render-dom-css-recalculate-style](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-dom-css-recalculate-style.png)

- HTML结合CSS重新计算样式

![brower-render-dom-css-to-renderTree](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-dom-css-to-renderTree.png)

- HTML与CSS结合起来，生成渲染树`render tree`，与`DOM tree`类似，但是有所不同
- 渲染数比起DOM树缺少某些内容，如`<HEAD>`,`<style>`,`<script>`等，头部与脚本等元素不见了

![brower-render-renderTree-only-display](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-renderTree-only-display.png)

- 值得一提的时，如果有元素的CSS属性有`display:none`，该节点的段落也不会出现在渲染树


![brower-render-tree](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-tree.png)

- 同样，如果CSS中添加了伪元素，如`after`,`befor`,就会添加到渲染树中

- 总而言之，只有实际上会显示在网页上的元素，才会出进入渲染树，课程的原话如下

> so this is essentially a simplified view of where the critical rendering path
> 所以这本质上是关键渲染路径的简化视图

## layout

布局

![brower-render-layout](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-layout.png)

- 这一步在chrome的devTool中显示为`Layout`, 也有称为`reflow`
- 布局，简单来说就是排版成一个个嵌套的方框 —— `网络布局模型`
- 这意味着某个元素可以影响到其他元素, 例如`body`的宽度通常会影响到子项的宽度等，一直往树的下方蔓延
- 值得注意的是，如果网页中执行`javascript`脚本修改了页面中`任意可显示元素`的`宽高`,不管这个元素是`body`还是某个毫不起眼的`div`，`scope`作用域都是`document`，都会导致整个`document`重新`layout` 

## raster 光栅化

![brower-render-raster](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-raster.png)

- 这一步是光栅，可以理解为将`矢量图`转化为`位图`
### raster for page 页面

![brower-render-raster-paint](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-raster-paint.png)


- 图中，左边是光栅器需要执行的的绘制调用，以便填充像素

![brower-render-raster-paint-complete](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-raster-paint-complete.png)

- 上图是填充完成的状态
- 填充顺序正如图左边的`绘制调用`如下
    + 绘制`文本`
    + 接着是`阴影`，`边框线条`，`图片`
    + 等等。。
- 这些步骤在`devTool`中显示为`Paint`

### raster for image 图片

![brower-render-raster-image-draw-bitmap](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-raster-image-draw-bitmap.png)

- 在前面的光栅调用中，可以看到`drawBitmap`
- 网页中出现图片时，浏览器需要请求图片文件，拿到之后，需要将图片文件的数据解码到内存，如下

![brower-render-raster-image-decode-resize](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-raster-image-decode-resize.png)

- 解码后，可能还需要调整尺寸

## layouts and composite

- 之前的`layout`布局与`raster`光栅步骤是处理单个`layer`图层，接下来是合成多个图层
- 在`devTool`中称作`composite layers`

![brower-render-layout-composite](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-layout-composite.png)

- 合成图层，影响的因素有：透明度，动画

## GPU put the pictures up on screen

![brower-render-layout-upload-gpu](https://cdn.jsdelivr.net/gh/lightfish-zhang/media-library/image/201707/brower-render-layout-upload-gpu.png)

- 最后GPU根据指示将图片显示到屏幕上

> In brief, it is how we get from a single request all the way through to pixels on screen
> 最后，这就是从单个请求一直到将像素填充到屏幕上的简单流程

# render pipline

渲染的管道, 这是渲染性能优化的重要知识点

- 当网页执行`javascript`脚本使页面发生变化时，可能执行以下步骤

    + `javascript` 通常，我们会使用javascript来处理内容并导致外观变化
    + `style` 如果通过CSS或js做出外观更改，浏览器必须重新计算受到影响的元素的样式
    + `layout` 如果改变了布局属性，即更改元素的几何结构，例如宽度高度，相对位置或者间距，那么浏览器需要检查所有其他元素并`re-flow the page`回流网页
    + `paint` 受到影响的区域需要重新绘制
    + `composite` 最后绘制的元素将需要合成在一起

- 不同的变化可能会执行不同的步骤，推荐网站[csstriggers.com][2]，它罗列出了不同的css样式修改影响到的不同的`渲染步骤`

# 附录

[1]:https://www.youtube.com/watch?v=hHvPD9m6ovM&index=2&list=PLAwxTw4SYaPl09X4Rljhy7dZinRCzbHz6

[2]:https://csstriggers.com/

[Browser Rendering Optimization 课程][1]

[csstriggers.com][2]