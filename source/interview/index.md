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

### 4. Web 性能优化策略(简述点)

主要从网络层面、浏览器渲染层面、服务端优化以及打包构建四个方面来讲，主要包括下边几点：

#### 网络

- DNS 缓存(一般都有)
- 资源缓存(配置正确的缓存策略，强缓存，协商缓存)
- 静态资源 CDN
- 减少 http 请求数量(合理合并 js,css,img 文件，资源懒加载(图片，单页应用组件)，内联部分 js, css 资源)
- 减少 http 请求大小(
  js: 代码压缩混淆，去语义化, 去注释
  css: 代码压缩
  img: 小图片合并或者内联资源
  )

#### 浏览器渲染

1. 减少回流和重绘
   - 减少回流：
   * 避免使用 table 布局
   * 使用文档碎片，避免多次操作 DOM
   * 避免在循环中获取元素尺寸信息
   * 设置样式时通过 class 来设置
   * 动画元素脱离文档流(absolute 或者 fixed 布局)
   * 动画元素自成一层(translate 取代 top 或者 marginTop.通过设置 3d 变换或者 will-change 等)
   - 减少重绘：
   * 使用 opacity 而不是 visibility: hidden;
2. css 资源位于 link 中，script 位于 body 底部（加快 DOM 树和 CSSOM 树的构建，加快渲染， script 部分资源可加 async 和 defer，不阻塞 DOM 树的构建）

#### 打包构建

- webpack 公共模块抽取(配置 optimize/commonChunks)
- 库代码采用 cdn 资源，不打包进业务包（或者单独打包）
- css,js uglify。
- 资源按需引入，分块打包, 避免主业务包过大。import
- tree shaking。通过静态分析，只打包用到的资源模块

#### 服务端

- 服务端首屏渲染(减少资源加载和请求，便于 seo，降低白屏时间)
- 合理设置缓存

---

### 5. absolute定位是相对于什么元素定位的？

当position设置为absolute时，是相对于其元素最近的非static模式的父元素进行定位的。

---

### 6. padding-top和margin-top为百分比时，是相对于什么取值的

两者都是相对于最近父级块状元素的width变化取值的。

---

### 7. cdn的查找过程

最简单的CDN网络是由一台DNS服务器和几台缓存服务器构成。

* 当用户请求资源时，经过DNS系统的解析，最终会将域名解析权交给网站的CNAME指向的CDN专用的DNS服务器。
* CDN专用的DNS服务器将全局负载均衡设备的IP返回给用户
* 用户向该IP发起请求，负载均衡设备会根据用户的IP地址，请求内容URL，选择一台离用户距离最近且有服务能力的缓存服务器，告诉用户向这台服务器发起请求。
* 用户接着向缓存服务器发送请求。缓存服务器收到请求，去查找对应的缓存内容。如果没有找到，会去向它的上级服务器或者到源服务器请求内容到本地，然后再返回给用户。

---

### 8. js中array底层是用什么数据结构实现的 

数组，是一种基础的数据结构。它从0开始，表示一段连续的内存结构，用来存储同类型的数据。

然而，js中的array与此不同的是，它可以存储不同类型的数据。number, string, object, array, symbol等都可以放在同一个数组中。

那么js中array底层到底是怎么实现的呢？
以chrome v8为例，在v8引擎中，数组的实现是基于快慢数组的。初始化一个数组中，如**var a = [];**，这时v8默认创建一个快数组(fastElement)，会去申请一段连续的内存。
当随着数组的push，数组内存空间被占满后，引擎会按照当前数组容量的1.5倍+16去进行扩容。
当数组pop时，如果当前数组的内存容量大于数组的length*2时，引擎会标记可以释放的内存空间，在gc中进行垃圾回收，释放内存。

那么什么时候用慢数组来存储呢？首先，慢数组是基于hashtable的。以哈希表的形式存储在内存中，它不需要连续的空间，但需要维护额外的哈希表。

慢数组和快数组是可以转化的。根据当前数组的具体情况采用更优的存储方案。
当新添加的index的值比现有数组的容量多1024及以上时，快数组转为慢数组结构。

---

### 9. cors设置的几个参数 

一般服务端设置CORS时，通常需要在响应头中设置这样几个参数来告诉浏览器正常返回数据：

* Access-Control-Allow-Origin: 允许跨域的域名
* Access-Control-Allow-Methods: 允许跨域的方法
* Access-Control-Allow-Credentials: 发送跨域请求是否发送COOKIE
* Access-Control-Allow-Headers: 首部字段用于预检请求的响应。其指明了实际请求中允许携带的首部字段。

在CORS中，分为复杂请求或者简单请求。

对于简单请求而言，它会直接向服务器发起资源请求。当method为:
* GET
* HEAD
* POST
且Content-Type为下列之一时(尤其是对于POST请求)：
* text/plain
* multipart/form-data
* application/x-www-form-urlencoded
时，为简单请求，会直接发起请求获取资源。

在METHOD是PUT，或者DELETE或者Content-Type是application/json时，视为复杂请求。对于复杂请求，需要先发送一个OPTIONS请求作为预检请求，获知服务器是否允许该实际请求。
当预检请求通过后，会再发送对应的真实请求给服务器去获取或者操作对应资源。

对于附带COOKIE的请求(即设置了withCredentials为true，以及设置了Access-Control-Allow-Credentials为true)的情况，Access-Control-Allow-Origin不得设置为*，需要指定为具体的域名。

---

### 10. last-modified作为缓存标识的缺点 

1. 对于1秒内的资源更新，无法更精确判断
2. 如果开发者先改了一个资源，后来又改了回去。那么访问第一次修改后资源的用户的缓存就失效了。其实文件内容没变，但是文件的更改时间变动了。在这种情况下其实用户的缓存还是有效的，但last-modified就没办法识别了。

基于此，引入了以文件内容为基础的ETAG缓存字段。

---

