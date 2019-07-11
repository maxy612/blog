---
title: Vue源码分析——响应式系统
date: 2019-07-11 22:34:17
tags: ['Vue']
summary: Vue中响应式系统是链接html模板与数据的核心枢纽，是数据驱动视图必不可少的一部分。今天就来讲讲响应式系统...
---

## 引言
在Vue中，我们的基本操作像下面这样：
``` javascript
<template>
<div id='app' @click='change'>
    <span>{{ msg }}</span>
</div>
</template>

<script>
export default {
    data() {
        return {
            msg: 'hello,world'
        }
    },
    
    methods: {
        change() {
            this.msg = 'abcdefg';
        }
    }
}
</script>
```

当我们点击div时，会发现span里边的内容由hello,world变成了abcdefg。这就是Vue响应式系统比较直观的展现了。我们只需要改变数据，就能在视图上看到这种改变。

### 抛开Vue，我们怎么样才能实现这样一个系统呢？即我只需要更新数据，就能直接在视图上看到变更后的数据......

基本思路其实也很简单。下面我们一步步来实现它。首先定义一段html模板。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta http-equiv="X-UA-Compatible" content="ie=edge" />
    <title>Static Template</title>
  </head>
  <body>
    <div class="app"></div>

    <script>
      window.onload = function() {
        const oApp = document.querySelector(".app");
        const data = {
          msg: ""
        };
        const Event = {
          subs: {},
          add(name, fn) {
            if (this.subs[name]) {
              this.subs[name].push(fn);
            } else {
              this.subs[name] = [fn];
            }
          },
          fire(name, data) {
            if (this.subs[name]) {
              this.subs[name].forEach(fn => {
                fn(data);
              });
            }
          }
        };

        const watcher = function(dom) {
          Event.add("change", function(data) {
            dom.textContent = data;
          });
        };

        Object.defineProperty(data, "msg", {
          get() {
            watcher(oApp);
            return this.value;
          },
          set(newVal) {
            this.value = newVal;
            Event.fire("change", newVal);
          }
        });

        // test
        data.msg = "maxy61";
        alert(data.msg);

        setTimeout(() => {
          data.msg = "maxy61209";
          alert(data.msg);
        }, 3000);
      };
    </script>
  </body>
</html>

```

具体的执行效果详见 [![Edit static](https://user-gold-cdn.xitu.io/2019/7/10/16bdc24d46eb5186?w=201&h=42&f=svg&s=21059)](https://codesandbox.io/s/static-z8nbe?fontsize=14)

#### 这里，我们实现了修改data.msg，div的内容随之变化的效果。但是和vue的那种令人舒服的调用模式还是天壤之别。不过尽管简单，但是也是Vue能够做到模板响应数据变化的基本原理。

*即通过Object.defineProperty来对数据进行拦截，拦截其get和set操作。在get时，收集关于访问该数据的dom模板信息；当数据改变时，调用之前添加的关于dom模板信息的函数。在Vue中，添加的是watcher实例。而和dom相关的watcher实例叫做渲染watcher。*

## Vue的响应式系统
Vue中的双向绑定的实现依赖于其响应式系统。而响应式系统的实现主要靠**数据劫持+发布订阅模式**来实现的。

在Vue2.0中，数据劫持主要依靠Object.defineProperty来实现的。而发布订阅模式主要表现在Dep和Watcher的关系上。具体代码如下：

```javascript
// 来自vue源码中 src/core/observer/index.js中
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

Vue的数据响应过程主要分为三步：1. defineReactive拦截get和set；2. 收集渲染watcher 3. 数据改变时触发渲染watcher，更新dom。
* 在Vue实例初始化时，会对配置项中options的data进行处理。对data进行遍历，然后对其中的每一项都用defineReactive拦截数据的get和set操作，同时为每个数据设置一个依赖收集器dep，来使其变成响应式属性。
* 在初始化的最后，会调用vm.$mount操作。该操作最终会生成一个渲染Watcher，渲染watcher在实例化的时候会调用watcher.get方法，于是Dep.target被设置为当前渲染watcher。之后调用vm._update(vm._render(), hydrate)。在这个过程中会触发对数据的访问，进而触发数据的get，于是当前的watcher就被添加到所访问数据dep的subs里。
* 当html模板所访问的数据发生变化时，数据的依赖收集器dep会调用notify方法，进而调用之前收集到的渲染watcher。渲染watcher最终再次调用vm._update(vm._render(), hydrate)方法，完成对html模板的更新。


**主要用到了三个构造函数：**
    Observer,
    Dep,
    Watcher

    Observer: 一个object对应一个Observer实例，对object下的每一项进行defineReactive（即数据劫持），使其变成响应式属性。
    Dep: 负责收集watcher及发布更新，和数据是一对一的关系
    Watcher: 负责具体的更新操作

## Vue数据劫持的缺陷

然而，Vue2.0的数据劫持也存在一定的问题。当数据是一个数组时，我有这样一段代码：
```javascript
<template>
  <div id="app">
    <img width="25%" src="./assets/logo.png">
    <p v-for='val in arr' :key='val'>{{val}}</p>
  </div>
</template>

<script>
export default {
  name: "App",
  data() {
    return {
      arr: [1, 2, 3, 4, 5]
    }
  },
  mounted() {
    setTimeout(() => {
      this.arr[0] = 10;
    }, 3000);
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
```

然而在mounted中的改变并没有任何效果，执行效果详见[codesandbox](https://codesandbox.io/s/vue-template-ygtyb)。Vue2.0对于数组的处理，是修改了Array原型上的几个方法，只有对数据的操作调用了对应的方法，才会具有响应式属性的效果。具体的方法有push, pop, shift, unshift, splice, sort, reverse.
当被观察的数组调用了上面这些方法时，才能够触发对应的视图更新。当然，如果数组中有object，当object变化时，视图也能够更新。

#### 鉴于当前数据劫持方案所暴漏出的缺陷，在Vue3.0中，尤大用ES6推出的Proxy来代替Object.defineProperty。
#### Proxy相对于Object.defineProperty，提供了更加全面的数据劫持能力。
详见[Proxy文档](http://es6.ruanyifeng.com/#docs/proxy)。

* 采用Proxy用作数据劫持后，我们可以对整个对象进行数据拦截，而不用再一个个定义属性的key去做拦截。不过如果对象下的属性是对象的话，需要对深层的数据再次生成Proxy实例。
* Proxy提供了get, set, apply, has, construct, deleteProperty, defineProperty等更多的数据劫持方法。
* 浏览器原生支持，无需像Object.defineProperty一样在处理数组时通过修改数组原型方法做hack。例如：[数组demo](https://codesandbox.io/s/static-e9rii)

-------------------------------------

大概就是这样了。
