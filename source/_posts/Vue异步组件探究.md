---
title: Vue异步组件探究
date: 2019-08-14 00:11:55
tags: ["Vue"]
summary: 前两天在整理项目的时候发现项目中用到了vue的异步组件，顺便对它做一番研究吧...
---

之前在回顾自己写的一个后台管理项目时，发现用到了Vue的异步组件。而之前恰好在研究vue的源码，顺便分析下异步组件的加载和执行过程。

### 异步组件是什么？

异步，是相对于同步而言的。我们在使用Vue时，使用到的组件大多为同步组件。在vue实例第一次执行渲染的过程中，已经生成了组件构造器。而异步组件则是在用到该组件时，异步通过请求去拉取对应组件的js（这里需要和webpack的import相结合）。当对应的js加载完成后，获取到异步组件的配置项，从而创建组件构造器。再通过forceRender，强制依赖它的vm实例重新触发渲染函数。

**为什么当时项目要选用异步组件呢？**

    因为对于后台管理系统来说，通常左侧有一系列的操作tab。而每个tab对应一个组件。
    当用户进入时，如果把所有的组件都进行加载，会导致用户体验下降，等待时间过长。所以就需要使用异步组件。

在用户进入页面时，只把它一开始用到的组件做成同步的。这样既不影响用户的操作体验，也优化了页面的加载速度。

> Vue中，如果有多个vm实例都用到了异步组件，异步组件只会加载一次，并在内存中做缓存，不会重复加载。


### 异步组件如何使用？
注册异步组件，可注册普通组件差不太多。既可以全局注册，也可以局部注册。不过不同的是异步组件需要通过webpack的import函数来引入。如下：
``` javascript
// 局部注册
new Vue({
    ...rest,
    components: {
        a: () => import('./components/a.vue')
    }
})

// 全局注册
Vue.component('async-comp', (resolve, reject) => ({
    component: () => imort('./components/a.vue'),
    loading: loadingComp,
    error: errorComp,
    delay: 200,
    timeout: 3000
}));
```

其中，异步组件还有高级异步组件。所谓高级异步组件，就是提供了loading组件(用于组件异步加载时展示)，error组件(用于组件加载出错时展示)，delay(用于延迟加载异步组件)，timeout(用于设置异步组件加载的最大时长，如果超过了规定的时间还没有加载完毕，则展示error组件)。

其中，error和loading组件都是同步的。因为当异步组件加载中或者加载出错时，需要立即展示对应的状态组件。

对于异步组件，通过使用webpack提供的import函数，可以让webpack在打包时将import中传入的组件文件单独打包。这样当使用到该组件时，通过发送请求异步加载组件的js，当资源加载完毕后交给vue来处理加载。

### 异步组件是怎样执行的？

首先，要先对webpack的import做一个说明：

    根据webpack官网的解释，import是用于动态代码拆分。它传入一个模块路径，然后返回一个promise实例。在webpack打包时，
    会将import引入的模块单独打包到一个代码包中。并且在代码执行到所在行时再去加载对应的资源。

对于webpack的import的探究，请参考[模块方法](https://webpack.docschina.org/api/module-methods#import-)。

下面来分析下异步组件究竟是怎么加载的。以上面用法中的全局异步组件为例：

1. Vue.component会在Vue.options.components上加入 {'async-comp': fn }。以便在组件渲染时执行vm._render方法时在createElement中找到对应的组件定义。
``` javascript
const ASSET_TYPES = ['component', 'directive', 'filter'];
ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
        // ... 略去和流程无关部分
        this.options[type + 's'][id] = definition
        // 执行过后，Vue.options.components['async-comp'] = fn
        return definition
      }
    }
  })
}
```
2. 在createElement中调用createComponent函数，进入组件vnode的创建过程。在createComponent中，对于异步组件的处理，主要是resolveAsyncComponent函数。
``` javascript
// 此处的调用关系：vm.$mount -> mountComponent -> vm._update(vm._render(), hydrating) -> 
// -> vm._render -> vm._c -> createElement -> _createElement -> createComponent
// 具体的流程就不具体分析了，最终会进入到createComponent函数中。

export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  // 此时 Ctor是之前定义的异步加载函数，并不是object
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }
  
  // 有删减

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    // 最终会进入这里。
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    // 第一次执行渲染函数时，如果不是高级异步组件或者高级异步组件的delay不为0，则ctor默认为undefined。返回一个空的占位节点
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  // ... 省略其余部分
  const vnode = new VNode(...opts);
  return vnode;
}
```
3. 在resolveAsyncComponent函数中，做了下面几件事：
    
    * 判断该异步组件的error及resolved，如果有error或isloading为true或者resolved不为undefined，则返回对应状态的组件。
    * 收集了引用该异步组件的所有的vm实例，并存储在factory.owners中。
    * 定义了forceRender函数(用于在加载成功或者失败后对owners遍历调用其渲染函数)，resolve(异步组件加载成功后的处理函数)，reject(异步组件加载失败后的处理函数)
    * 调用异步组件对应的函数，并对高级异步组件做进一步的处理。
    
``` javascript
export function resolveAsyncComponent (
  factory: Function,
  baseCtor: Class<Component>
): Class<Component> | void {
  /** 第二次进入时会命中 start */
  // 如果异步组件加载失败或者超时(timeout)了，会进入这里
  if (isTrue(factory.error) && isDef(factory.errorComp)) {
    return factory.errorComp
  }

  // 组件被正常加载后，会进入这里
  if (isDef(factory.resolved)) {
    return factory.resolved
  }
  
  // 收集引用该异步组件的vm实例，以便在forceRender时触发对应vm实例的渲染函数
  const owner = currentRenderingInstance
  if (owner && isDef(factory.owners) && factory.owners.indexOf(owner) === -1) {
    // already pending
    factory.owners.push(owner)
  }

  // 如果处于loading状态，返回loadingComp。
  if (isTrue(factory.loading) && isDef(factory.loadingComp)) {
    return factory.loadingComp
  }
  /** 第二次进入时会命中 end */

  if (owner && !isDef(factory.owners)) {
    const owners = factory.owners = [owner]
    let sync = true
    let timerLoading = null
    let timerTimeout = null

    // 当vm实例销毁时，将factory.owners中当前监听的owner移除。
    ;(owner: any).$on('hook:destroyed', () => remove(owners, owner))

    const forceRender = (renderCompleted: boolean) => {
      for (let i = 0, l = owners.length; i < l; i++) {
        // 重新执行了vm渲染watcher的update方法，即vm._update(vm._render(), hytrating)
        (owners[i]: any).$forceUpdate()
      }

      if (renderCompleted) {
        owners.length = 0
        if (timerLoading !== null) {
          clearTimeout(timerLoading)
          timerLoading = null
        }
        if (timerTimeout !== null) {
          clearTimeout(timerTimeout)
          timerTimeout = null
        }
      }
    }

    const resolve = once((res: Object | Class<Component>) => {
      // 缓存并生成加载回来的异步组件构造器
      factory.resolved = ensureCtor(res, baseCtor)
      // invoke callbacks only if this is not a synchronous resolve
      // (async resolves are shimmed as synchronous during SSR)
      // 当异步组件加载完成时，此时sync为false。执行forceRender触发重新渲染，再次进入该函数后命中factory.resolved分支。
      if (!sync) {
        forceRender(true)
      } else {
        owners.length = 0
      }
    })

    const reject = once(reason => {
      process.env.NODE_ENV !== 'production' && warn(
        `Failed to resolve async component: ${String(factory)}` +
        (reason ? `\nReason: ${reason}` : '')
      )
      // 加载异步组件失败的情况
      if (isDef(factory.errorComp)) {
        factory.error = true
        forceRender(true)
      }
    })
    
    // 对于高级异步组件，会返回一个object，如示例用法中全局注册组件。
    const res = factory(resolve, reject)

    if (isObject(res)) {
      // 对于上述用法中 局部注册组件那样, import会返回一个promise实例。
      if (isPromise(res)) {
        // () => Promise
        if (isUndef(factory.resolved)) {
          res.then(resolve, reject)
        }
      } else if (isPromise(res.component)) {
        // 高级异步组件。例如实例中配置了 component: () => import('./a.vue')
        res.component.then(resolve, reject)
        
        // 缓存错误组件构造器
        if (isDef(res.error)) {
          factory.errorComp = ensureCtor(res.error, baseCtor)
        }
        
        // 缓存loading组件构造器
        if (isDef(res.loading)) {
          factory.loadingComp = ensureCtor(res.loading, baseCtor)
          if (res.delay === 0) {
            // 只有delay配置为0时，第一次执行渲染时才会返回loading组件构造器
            factory.loading = true
          } else {
            // 如果设置了delay, 
            timerLoading = setTimeout(() => {
              timerLoading = null
              if (isUndef(factory.resolved) && isUndef(factory.error)) {
                factory.loading = true
                forceRender(false)
              }
            }, res.delay || 200)
          }
        }

        if (isDef(res.timeout)) {
          // 如果异步组件加载时间超过了设置的timeout，也视为组件加载失败
          timerTimeout = setTimeout(() => {
            timerTimeout = null
            if (isUndef(factory.resolved)) {
              reject(
                process.env.NODE_ENV !== 'production'
                  ? `timeout (${res.timeout}ms)`
                  : null
              )
            }
          }, res.timeout)
        }
      }
    }

    sync = false
    // return in case resolved synchronously
    return factory.loading
      ? factory.loadingComp
      : factory.resolved
  }
```
    
所以异步组件的加载和执行实际上是通过执行两次渲染函数实现的。

第一次执行的时候，对异步组件对应的函数进行处理，定义处理loading, error状态组件，并给factory传入resolve, reject。当异步组件文件加载完成后通过resolve或者reject函数进行相应的处理，此时factory的error或者resolved状态会发生变化。设置完factory.error或者factory.resolved后，会调用forceRender，触发对应vm实例的渲染watcher的update，重新执行渲染，进而执行第二次渲染。

第二次渲染时，同样会进入resolveAsyncComponent方法中。此时异步组件的加载结果已经确定了，会返回对应的成功状态组件(即我们定义的异步组件)或者失败状态的组件。之后接着执行后续流程。

-----------------------

对异步组件的探究就先到这里了...