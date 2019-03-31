---
title: 记fullpage使用中的一个小bug
date: 2018-03-29 22:10:56
tags: ["fullpage"]
summary: 之前同事在用fullpage做愚人节的一个h5，在经历了vh和100%的问题以及样式问题和机型兼容问题后，又遇到了fullpage的touchmove问题。下面简单来介绍一下。
category: "实战"
---

###
1. fullpage简介
2. fullpage测试过程中遇到的问题
3. fullpage问题的解决方案

----------------------------

## fullpage简介
fullpage是一个功能强大的基于jquery的一个像ppt那样的全屏滚动插件。无论是在pc端还是移动端，它都在被广泛使用着。使用fullpage，我们只要按照fullpage规定的dom及其属性命名方式就可以快速制作出一个全屏滚动的h5，在宣传和引流方面起着重要作用。

## fullpage测试过程中遇到的问题
在介绍这个问题之前，先说一个之前同事在处理h5页面时遇到的一个问题。为了做到全屏展示，同事把包含滚动代码的父级直接设置了100%，高度为100vh。在安卓上测试页面显示正常。后来在苹果的safari下测试，每屏下方总有一部分被遮盖。想起之前用h5哄前前女友的装逼失败的惨痛教训，建议他改成100%，然后就好了。后来我又测试了一下，最后发现safari中确实存在100vh比安卓机的高出一部分。

接下来进入正题。在历经千辛万苦做完了h5的页面后，提交到测试。结果出现滑动一次可以滑动多页的情况。如果滑动过程中手指未抬起，会不断地进行页面滑动。作为实习生，我刚好手里没有活儿，于是帮着看了看。

遇到这个问题，我第一个想到的就是fullpage的touchmove事件的问题，究竟是由于同事自己的代码导致的还是fullpage本身的问题呢？于是我自己做了一个简易版的fullpage。

{% img /images/fullpage.gif [400] [title fullpage] %}

如图所示，即使是未加修饰的fullpage，也存在这个问题，可见是fullpage本身的问题。为了彻底解决这个问题，有必要翻一番fullpage的源码来看一下。

{% img /images/events.png [400] [title 定义的事件] %}
（定义的事件）

{% img /images/touchhandler.png [400] [title 添加触摸事件] %}
（添加触摸事件）

{% img /images/touchstart.png [400] [title touchstart] %}
(touchstart的处理事件)

{% img /images/touchmove.png [400] [title touchmove] %}
（touchmove的处理事件）

从源码中可以看到，在传入的父容器上只绑定了touchstart和touchmove两个事件，没有添加对touchend事件的监听，在触发touchmove的时候就会触发滚动事件。这就解释了为什么手指滑动后未抬起或者滑动动作过大时就会出现一次滑动多张的情况。

## fullpage问题的解决方案
在fullpage中滑动与否是需要通过slideMoving这个参数来控制的。当slideMoving为false时会进入touchmove的监听事件里的滑动部分，滑动之前将slideMoving设置为true,在滑动完成后将slidMoving设置为false。原有的设计并没有考虑到touchend事件，所以会导致上述问题。所以在这里，我尝试通过一个参数来控制它的滚动与否，我将其命名为isSliding，初始值为false。当进入touchmove的处理事件中，我会先判断isSliding是否为true，如果是true，代表它的手指并未抬起触发touchend事件，则返回，不做任何处理。如果是false,则进入处理逻辑中。
然后在原有的参数slideMoving设置为true的地方增加一行 isSliding = true。然后监听touchend事件，在touchend中将isSliding设为false。这样就避免了一次滑动多张以及手指未抬起导致一直滑动的问题。代码如下：

{% codeblock lang:javascript %}
    var FP = $.fn.fullpage; // 101行
    var isSliding = false; // 102行  定义一个isSliding的变量

    // ......

    // 206 - 210
    var events = {
        touchmove: 'ontouchmove' in window ? 'touchmove' :  MSPointer.move,
        touchstart: 'ontouchstart' in window ? 'touchstart' :  MSPointer.down,
        touchend: 'ontouchend' in window ? 'touchend' : MSPointer.up
    };

    // 2581 - 2584 addTouchhandler函数中
    $(WRAPPER_SEL)
        .off(events.touchstart).on(events.touchstart, touchStartHandler)
        .off(events.touchmove).on(events.touchmove, touchMoveHandler)
        .off(events.touchend).on(events.touchend, touchendHandler);

    // 1113 touchmoveHandler首行
    if (isSliding) return;

    // 1312 moveSlide函数中
    isSliding = true;

    // 新增一个touchendHandler事件
    function touchendHandler() {
        isSliding = false;
    }
{% endcodeblock %}

另外，推荐一个生成gif的好工具gif5,支持视频和图片生成，真的是非常方便哦...