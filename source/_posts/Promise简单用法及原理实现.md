---
title: Promise简单用法及原理实现
date: 2017-10-18 15:21:30
tags: ["Promise"]
summary: Promise是ES6中引入的用于解决异步的一个方法，有了它，我们可以以同步的方式写异步代码，避免了回掉地狱，代码上也更加简洁。掌握Promise非常重要，在面试笔试中也会被问到，在本文中我将介绍Promise的简单用法和原理实现，希望对大家有用。
category: "学习"
---

作为Promise的入门级文章，本文主要将一下几个方面：
1. Promise是什么？
2. 为什么要用Promise呢？
3. Promise的简单用法
4. Promise的原理实现
5. Promise的延伸

- - - - - - - - - - - -
## Promise是什么？

Promise对象用于一个异步操作的最终完成（或失败）及其结果值的表示。通俗地讲，Promise可以对异步事件进行封装，以同步的形式处理回调函数。它简化了异步编程，以一种更容易让人理解的方式处理异步回调，同时对于排查错误也非常有用。

之前在做百度的秋招笔试题时，有一个操作题是让手写Promise的实现，当时什么都不记得，只是用ES6搭了个架子就灰溜溜地交卷了。后来，我找了一些网上的Promise原理实现来看，似懂非懂。前后大概看了有四五个实现吧，终于有点眉目了，尝试自己实现一个Promise，还是不行。上周五的时候电话面试了一个公司的前端，被问到手动实现Promise，只能暂时搪塞过去了。今天抽空又看了一遍Promise，觉得还是要把这块儿硬骨头啃下来。

- - - - - - - - - - - -
## 为什么要用Promise呢？

看下面一ajax请求的例子：
##### a.异步回调
以前，我们封装调用一个异步请求，一般是这样搞得：
{% codeblock lang:javascript %}
function ajax(json) {
  var url = json.url || '',
      method = json.method || 'get',
      async = json.async || true,
      withCredentials = json.withCredentials || false,
      succfn = json.success || function () {},
      data = json.data || {},
      failfn = json.fail || function () {}, xhr;

  if ('XMLHttpRequest' in window) {
    xhr = new XMLHttpRequest();
  } else {
    xhr = new ActiveXObject('Microsoft.XMLHTTP');
  }

  var dataStr = '';
  for (var key in data) {
    if (data.hasOwnProperty(key)) {
      dataStr += key + '=' + encodeURIComponent(data[key]) + '&';
    }
  }

  dataStr = dataStr.substring(0, dataStr.length - 1);

  xhr.withCredentials = withCredentials;
  if (method.toLowerCase() === 'get') {
    xhr.open(method, url + '?' + dataStr, async);
    xhr.send();
  } else if (method.toLowerCase === 'post') {
    xhr.open(method, url, async);
    xhr.send(dataStr);
  }

  xhr.onreadystatechange = function () {
    if (xhr.readyState == 4) {
      if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304) {
        succfn && succfn(xhr.responseText);
      } else {
        failfn(xhr.status);
      }
    }
  }

  xhr.onerror = function (e) {
    failfn(e);
  }
}

// 调用时，
ajax({
    url: '',
    method: '',
    data: {},
    success: function (res) {},
    fail: function (e) {}
})
{% endcodeblock %}

对于单个的请求，足够用了，但是如果我们要发送好几个请求，且请求之间有数据依赖性的关系时，就会很麻烦（比如先发送一个请求获取用户id，然后通过用户id再次发送请求，获取用户手机号）。

{% codeblock lang:javascript %}
ajax({
    url: 'http://xxxx.com/api/getUserId',
    method: 'get',
    data: {},
    success: function (res) {
      ajax({
          url: 'http://xxxx.com/api/getPhone',
          data: {
            userid: res
          },
          success: function (res) {}
      })
    },
    fail: function (e) {}
})
{% endcodeblock %}

这只是两个有依赖请求时怎样处理回调，如果是有5，6个请求呢，那下一个请求就必须在前一个ajax请求的success的回调中来设置，这就出现了所谓的回调地狱，回调的层次会非常深，一旦出错，很难排查问题所在。这时，我们的Promise登场了，来拯救被回掉地狱困着的我们......

{% codeblock lang:javascript %}
// 我们借用之前的ajax封装函数，
function ajax(json) {
  // 重点1
  return new Promise(function (resolve, reject) {
  // 重点1结束
    var url = json.url || '',
        method = json.method || 'get',
        async = json.async || true,
        withCredentials = json.withCredentials || false,
        data = json.data || {};

    if ('XMLHttpRequest' in window) {
      xhr = new XMLHttpRequest();
    } else {
      xhr = new ActiveXObject('Microsoft.XMLHTTP');
    }

    var dataStr = '';
    for (var key in data) {
      if (data.hasOwnProperty(key)) {
        dataStr += key + '=' + encodeURIComponent(data[key]) + '&';
      }
    }

    dataStr = dataStr.substring(0, dataStr.length - 1);

    xhr.withCredentials = withCredentials;
    if (method.toLowerCase() === 'get') {
      xhr.open(method, url + '?' + dataStr, async);
      xhr.send();
    } else if (method.toLowerCase === 'post') {
      xhr.open(method, url, async);
      xhr.send(dataStr);
    }

    xhr.onreadystatechange = function () {
      if (xhr.readyState == 4) {
        if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304) {
          // 重点2
          resolve(xhr.responseText);
          // 重点2 结束
        } else {
          // 重点3
          reject(xhr.status);
          // 重点3 结束
        }
      }
    }

    xhr.onerror = function (e) {
      // 重点4
      reject(xhr.status);
      // 重点4 结束
    }    
  })
}
{% endcodeblock %}

改动很少，只是加入了Promise的部分。现在，当我们之前两个请求时，可以这么做：

{% codeblock lang:javascript %}
ajax({
  url: 'http://xxxx.com/api/getUserId',
  method: 'get',
  data: {}
}).then(res => {
  return ajax({
    url: 'http://xxxx.com/api/getPhone',
    data: {
      userid: res
    }
  })
}).then(res => {
  console.log(res);
}).catch(e => {
  console.error('err: ', e);
})
{% endcodeblock %}

这样，我们用Promise成功的处理了回调地狱的问题，如果之前的哪个步骤出错了，最后的catch都会捕获到错误的。当然，这只是Promise的一个用法。

##### b.请求并发
除此之外，如果我们想发送一个请求，但是请求所需要的数据需要发送几个互不依赖（假设为a1, a2, a3, a4）的请求来获取，如果用Promise呢？很简单,直接用Promise.all[ajax(a1), ajax(a2), ajax(a3), ajax(a4)]).then(values => {console.log(values);})就好了，关于Promise.all,后边会详细说明。

##### c.请求最优策略
如果我们有两个以上功能类似的接口在不同的服务器上且都可用，要获得对应的数据，可以单独获取一个请求，但我们想请求优化一些，就是向服务器分别发送请求，取先返回的请求的数据，这时我们可以用Promise.race([ajax(a1), ajax[a2]]).then(value => {console.log(value);}), 我们将会在第一时间内获得返回的数据。

当然，Promise还有很多用法，我这里只是列出了一部分，有兴趣的话可以继续探索。
- - - - - - - - - - - - - - - -
## Promise的简单用法

下面，介绍一下Promise的基础知识。

Promise有pending, resolved, rejected三种状态，可用的状态转换是:

##### status
pending -> resolved
pending -> rejected

状态一旦改变，以后该Promise的状态将不会变化。同时，执行对应的处理事件。

##### then
then方法在使用时，传入处理成功和失败的函数，当状态发生变化时，会调用对应的函数进行处理，然后返回一个值或者新的Promise。

##### catch
相当于then(undefined, fn), 可以捕获异步操作中的错误。如果有多个then链式调用，在多个then之后加上catch，将会捕捉到Promise中的错误，返回一个新的Promise。

##### race和all
当多个Promise同时执行时，如果其中一个Promise的状态由pending变为resolved或rejected，则Promise.race([])将会返回最先改变状态的Promise并执行对应的resolve或者reject。可以把race的数组参数中的多个Promise看作几个长跑运动员，谁先到达终点，就选择谁。
Promise.all与Promise.race相反，当Promise.all([])参数中所有Promise的status都变为resolved时，才会返回一个新的Promise，而数组中Promise的resolve的结果作为参数组成一个结果数组，在新的Promise执行resolve时传入。相反，当数组中的Promise中任何一个状态变为rejected，则整个Promise.all调用会立即终止，并返回一个reject的新的Promise对象。

使用Promise时，我们需要初始化一个Promise实例；

{% codeblock lang:javascript %}
let fs = require('fs');

var readFile = new Promise(function (resolve, reject) {
  // 这里放一些异步操作，如
  fs.readFile(url, (err, data) => {
      if (err) {
        reject(err); // 如果读取文件失败了，我们调用reject,并将err错误信息传入。
      } else {
        resolve(data); // 将读取成功的信息作为参数传入
      }
  })
})

readFile.then(data => {
  console.log(data); // 接收读取的数据进行操作
}).catch(e => {
  console.error('err: ', e); // 接收出错信息并进行处理
})
{% endcodeblock %}

初始化一个Promise实例，除了new Promise, 还有简便的方法。

{% codeblock lang:javascript %}
var a = Promise.resolve(4);
a.then(val => {
  console.log(val); // 4
})

var b = Promise.reject(Error('哈哈，出错了'))
b.catch(e => {
  console.log('err: ', e); // err:  Error: 哈哈，出错了
})
{% endcodeblock %}

通过调用Promise.resolve或者Promise.reject，也会返回一个新的Promise实例。
Promise.resolve还有拆包的功能，即如果参数是一个Promise,会直接返回参数，作为参数的Promise的resolve或reject。

{% codeblock lang:javascript %}
Promise.resolve(new Promise((resolve, reject) => {
  resolve(4);
})).then(val => {
  console.log(val); // 4
})

Promise.resolve(new Promise((resolve, reject) => {
  reject(Error('又出错了'));
})).catch(e => {
  console.log('err: ', e); // err:  Error: 又出错了
})

Promise.reject(Error('错错错'))
.catch(e => {
  console.log('err: ', e); // err:  Error: 错错错
})

// reject 不会解析参数里的Promise，它会将里边的Promise当作一个整体进行处理
Promise.reject(new Promise((resolve, reject) => {
  resolve(4);
})).catch(e => {
  console.log('err: ', e); // err:  Promise { 4 }
})
{% endcodeblock %}

还是控制一下篇幅，原理实现部分就在下篇文章中来做吧

......未完待续
