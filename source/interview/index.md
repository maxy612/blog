---
title: 面试题系列
date: 2020-05-25 19:41:32
---

### 1. https 比 http 安全在哪

详见[https 比 http 安全在哪](http://maxy612.cn/2020/05/25/https%E4%B8%BA%E4%BB%80%E4%B9%88%E6%AF%94http%E6%9B%B4%E5%AE%89%E5%85%A8/)

---

### 2. mouseover 和 mouseenter 区别

mouseover: 鼠标移入事件所绑定的元素及其子元素时触发
mouseenter: 鼠标移入事件所绑定的元素时触发（不含子元素）
当在一个父元素上添加 mouseover 事件和 mouseenter 事件，分别打印 mouseover 和 mouseenter。在父元素中有一个子元素。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Static Template</title>
    <style>
      #parent {
        width: 300px;
        height: 300px;
        border: 1px solid #f00;
      }

      #child {
        width: 100px;
        height: 100px;
        margin-left: auto;
        margin-right: auto;
        background: #ff0;
      }
    </style>
  </head>
  <body>
    <div id="parent">
      <div id="child">child</div>
    </div>

    <script>
      const parent = document.querySelector("#parent");
      const child = document.querySelector("#child");

      parent.addEventListener(
        "mouseover",
        () => {
          console.log("mouseover");
        },
        false
      );

      parent.addEventListener(
        "mouseenter",
        () => {
          console.log("mouseenter");
        },
        false
      );
    </script>
  </body>
</html>
```

此时，当鼠标由外界划入 parent 然后再划入 child，再划出 child，再划出 parent。输出顺序如下：

```javascript
// mouseover mouseenter 鼠标划入parent
// mouseover 鼠标由parent划入child
// mouseover 鼠标由child划入parent
```

---

### 3. "script" 标签中 "async" 和 "defer" 的作用

在浏览器解析 html 文档构建 dom 树的过程中，如果遇到了 script 标签，正常情况下会暂停 dom 树构建，去请求 script 中的资源并且执行，这样会阻塞 dom 树的构建进而影响渲染。

async 和 defer 表示该 script 中的资源为非关键资源，使得浏览器异步加载对应的 script 资源，降低对 dom 树构建过程的阻塞。

不同的是，async: 异步加载 js 资源，加载完之后立即执行；defer: 异步加载 js 资源，在 DOMContentLoaded 事件之前执行

---
