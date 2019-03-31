---
title: css中float研究
date: 2018-12-08 23:53:13
tags: ["学习", "CSS", "前端"]
category: "学习"
summary: 之前对css中的某些概念不太理解，就买了本《cssworld》来看。刚好翻到了float这章，看完顺便检查下自己都记住了点什么吧...
---

本来应该昨天写这篇的，可是昨天回来啥都不想干，直接玩手机玩到了深夜，罪过罪过...

下面我们来讲讲css中的float

顾名思义，float，译为浮动。是css的一个样式。它的主要作用是实现文字环绕图片效果，即图文混排，后来也被用在布局上。

### float的由来

float为什么会出现呢？html和css出现之初，只有为数不多的几个属性。页面的排版布局也很有限。后来，<b>为了实现在网页中展示文字环绕图片效果，出现了float</b>。它是css2的属性。

### float的作用

用过float的人都知道，它可以实现诸如[图文混排](https://codepen.io/maxycode/pen/wRwLwm),[页面布局](https://codepen.io/maxycode/pen/MZgMWx)等效果。


### float的原理

那么float为什么能够做到文字环绕呢？
没有浮动以前，页面中的元素要么横排，要么纵排。那么浮动怎么是怎么做到了呢？利用了两点：
1. 浮动元素使父元素高度塌陷
2. 行框元素不与浮动元素重叠

第一点使得浮动元素与文字能够处于同一水平线上，第二点使得浮动元素不会覆盖同一水平线的行框元素，两点结合从而形成了图文混排。

然而呢，后来在css3出现以前，开发者常常把浮动用在布局上，所以初级开发者经常会受到浮动元素父元素高度塌陷问题的困扰，便有了清除浮动的解决办法。

清除浮动的原理也不难，就是利用bfc(块级格式化上下文)可以包含浮动元素的特性来实现的。

在bfc元素中，无论子元素如何，都不会影响bfc同级元素造成影响。例如：[bfc特性](https://codepen.io/maxycode/pen/NeWdqM)

触发bfc的方式有很多：
1. 根元素自带bfc
2. position不为static, relative, sticky的元素
3. float不为none
4. display为inline-block, table的元素
5. overflow不为visible和inherit的元素

就先说到这里啦，拜拜