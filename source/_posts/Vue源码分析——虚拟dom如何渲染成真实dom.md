---
title: Vue源码分析——虚拟dom如何渲染成真实dom
date: 2019-07-21 23:02:02
tags: ['Vue']
summary: 虚拟dom是vue实现渲染优化所用的策略。那么从虚拟dom到真实dom都发生了什么呢，今天来探究一下...
---

今天我们来说下vue实例的\$mount中都发生了什么。\$mount是Vue原型上的方法，是Vue实例化的最后一步。$mount分为带编译器版本和不带编译器版本。我们以下面的代码为例，来讲下在\$mount时都发生了什么。

实例代码如下（来源于codesandbox的默认vue项目代码）：
``` javascript
// main.js
import Vue from "vue";
import App from "./App.vue";

Vue.config.productionTip = false;

new Vue({
  render: h => h(App)
}).$mount("#app");

// app.vue
<template>
  <div id="app">
    <img width="25%" src="./assets/logo.png">
    <HelloWorld msg="Hello Vue in CodeSandbox!" />
  </div>
</template>

<script>
import HelloWorld from "./components/HelloWorld";

export default {
  name: "App",
  components: {
    HelloWorld
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

// HelloWorld.vue
<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <h3>Installed CLI Plugins</h3>
  </div>
</template>

<script>
export default {
  name: "HelloWorld",
  props: {
    msg: String
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
```

**$mount是vue实例化中不可缺少的一部分，它将template转化成虚拟dom，然后根据虚拟dom渲染真实dom节点并进行挂载。在这个过程中，主要讲以下几点：**

* mountComponent中都发生了什么
* 渲染watcher(renderWatcher)的执行过程
* entry-runtime.js和entry-runtime-with-compiler.js的区别及适用场景 

---------------------

### 1. mountComponent中发生了什么？
在Vue实例化完成的最后一步(即_init原型方法中)，如果初始化的参数里有el，则自动调用$mount方法。否则，需要手动调用\$mount方法并传入挂载节点。

实例代码中，是在new Vue()之后手动调用了$mount方法，并传入了App组件。下面找到\$mount方法的定义：

``` javascript
// src/platforms/web/runtime/index.js
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  
  /** 
   * el: {name: "App", components: {…}, render: ƒ, staticRenderFns: Array(0), _compiled: true, ...rest}
   **/
  return mountComponent(this, el, hydrating)
}
```

由上可知，$mount获取到传入的App, 并且去query。在query中，由于App经过vue-loader编译后是一个Object，所以在query中被直接返回了。所以然后紧接着调用mountComponent函数。

``` javascript
// src/core/instance
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  // 1. 检查vm.$options.render是否存在，如果不存在，给出警告
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  
  // 2. 调用beforeMount生命周期钩子
  callHook(vm, 'beforeMount')

  let updateComponent
 
  updateComponent = () => {
    // mountComponent的核心
    vm._update(vm._render(), hydrating)
  }

  // 3. 生成渲染watcher
  new Watcher(vm, updateComponent, noop, {
    before () {
      // 如果当前vm实例已经被挂载，并且没有被销毁，调用beforeMount生命周期钩子
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```

在mountComponent中，renderWatcher是最关键的地方。通过renderWatcher，将render函数转换成vnode，然后通过vm._update，将vnode渲染成真实的dom。

-----------------------

### 2. 渲染watcher(renderWatcher)的执行过程

渲染watcher的执行过程，就是Watcher实例化的过程。
``` javascript
// src/core/observer/watcher.js
export default class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    // 当前isrenderwatcher为true
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.before = options.before // 这个函数会选择性调用beforeUpdate钩子函数
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }

    // expOrFn 就是updateComponent函数。
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn // 会走到这里啦
    }
    
    this.value = this.lazy // lazy是false，所以会调用this.get函数。
      ? undefined
      : this.get()
  }
  
  get () {
    pushTarget(this) // 将当前的Dep.target设置为该renderWatcher。
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm) // 在这里会调用updateComponent，即vm._update(vm._render(), hydrating);
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      if (this.deep) {
        traverse(value)
      }
      popTarget() // getter执行完毕后把Dep.target设置为之前的watcher实例。
      this.cleanupDeps()
    }
    return value
  }
}
```
> 由以上代码可知，在renderWatcher的执行过程中，this.get()的执行是最为重要的。而this.get()的执行则主要切换了Dep.target，执行了updateComponent函数。

updateComponent中, 主要执行了vm._update(vm._render(), hydrating)函数。下面着重来分析vm._update的过程。

----------------------------------

### vm._render的执行过程

_render是Vue原型上的方法，定义在src/core/instance/render.js中，下面分析下_render方法主要做了什么。
其实_render主要做了一件事，调用render函数生成vnode(虚拟dom)并返回。

    1. 从vm.$options取出render和_parentVnode。在上边的例子中，new Vue({ render: h => h(App) }).$mount('#app')。
    在这里, _parentVnode为undefined。
    2. 设置_parentVnode为vm.$vnode，即当前vue实例的占位符vnode。
    3. 设置currentRenderingInstance = vm, 同时执行渲染函数render，取得render返回的vnode。
    4. 然后将currentRenderingInstance置为null
    5. 返回vnode，并设置vnode.parent为_parentVnode.

在这里render函数主要调用了 render.call(vm, vm.$createElement)。对应于例子中, h就是vm.$createElement。就是根据传入的tag或者component，生成对应的vnode的一个函数。下面来分析下vm.$createElement函数。 vm.$createElement也定义在render.js文件中，在vm._init中执行挂载。vm.$createElement定义如下：

``` javascript
// 最后一个true用以标示是用户主动调用，即定义了render函数。而非由编译器调用。
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

// 在例子中，传入的参数如下：
vm.$createElement(App);
```

createElement最终会调用_createElement，下面来分析下_createElement。

``` javascript
// src/core/vdom/create-element.js
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  // 如果传入的data是一个响应式数据，直接创建并返回空vnode
  if (isDef(data) && isDef((data: any).__ob__)) {
    return createEmptyVNode()
  }
  // 处理<component is=""></component>动态组件情况
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    return createEmptyVNode()
  }
 
  // ... 有删减
  
  // 在例子中，这两个值是相等的。处理过程在createElement中。
  if (normalizationType === ALWAYS_NORMALIZE) {
    // 处理children, 最终生成一个[vnode, vnode, vnode]数组。
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  // 此时tag为对象App
  if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    // 直接进入到这个分支
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

接下来会调用createComponent函数来创建组件。在createComponent中，主要做了下面几件事：

``` javascript
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  const baseCtor = context.$options._base
  
  // 1. 调用Vue.extend创建子组件构造器VueComponent
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  data = data || {}

  // 2. 同步全局的属性到子组件构造器，主要是处理全局的mixins
  resolveConstructorOptions(Ctor)

  // 3. 安装组件管理的钩子init, prepatch, insert, destroy. 这些钩子将在不同的时期调用。
  installComponentHooks(data)
  
  // 4. 创建vnode并返回
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  return vnode
}
```

至于Vue.extend及其它函数的细节，就不详细讲了。大概明白它主要做了什么就好。

**至此，vm._render的过程就进行完了，主要就是根据render函数创建并返回了vnode。**
________________________________

### vm._update过程

vm._update主要是将vnode转换成真实dom。在这个过程中，包含了真实dom节点的增删改查操作，dom节点属性的操作及虚拟节点vnode的patchDiff过程。

vm._update也是Vue原型上的方法，定义在src/core/instance/lifecycle.js中。在_update中，主要就是取出之前的渲染vnode, 然后新旧vnode进行patch。在patch中将vnode的改动映射到真实dom上。在这个过程中，尤其是新旧vnode都存在时，通过patchDiff算法，找出最小的更改点，尽可能重用dom或者尽可能少地操作dom，达到一个相对较好的效果。

在patch的过程中，activeInstance为当前vm实例。当patch过程完成后，还原activeInstance为之前的vm实例。

第一次进行渲染时，preVnode为undefined。所以会进入if分支。

``` javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    // 1. 设置当前vm为activeInstance实例，在patch过程中会用到
    const restoreActiveInstance = setActiveInstance(vm)
    // 2. 设置vm的渲染vnode为传入的vnode
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // 3. 在上边的例子中，初次渲染，会进入这个分支
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    
    // 4. 还原之前的vm实例为activeInstance
    restoreActiveInstance()
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // 高阶组件相关
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
  }
```

**vm.__patch也是Vue原型上的方法。__patch__最终调用了src/core/vdom/patch中的createPatchFunction方法，它的返回值就是__patch__函数。**

> 在调用createPatchFunction时，传入了nodeOps和modules。nodeOpts和modules都是平台相关的。这也就是为什么Vue能在多个平台上工作的原因。以weex为例，Vue负责生成vnode，在createPatchFunction时传入平台相关的节点操作和modules，提前固化相关的节点操作和模块配置，以便在patch的过程中生成平台相关的节点和进行相关的节点操作。

    在createPatchFunction中，提取了modules中的'create', 'activate', 'update', 'remove',
    'destroy'的钩子函数，并存储在cbs中，以便后期的patch函数调用。最后，返回了patch函数。
    

最终工作的还是createPatchFunction返回的patch函数。这是整个patch过程的核心。下面来看下这个patch函数。

``` javascript
return function patch (oldVnode, vnode, hydrating, removeOnly) {
    // 如果新vnode不存在，旧的存在，调用钩子函数销毁旧的vnode
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // 如果旧的vnode不存在，创建vnode
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // 真正的update，新旧vnode对比更新
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // 初次patch。 例子中会进入这里, 创建一个空的vnode。
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm // 即$mount('#app')中的#app的dom节点
        const parentElm = nodeOps.parentNode(oldElm) // body

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm) // textnode
        )

        // update parent placeholder node element, recursively
        // 由于示例代码中，vnode的parent为undefined，所以不会进入该分支。
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
```
 
 对于示例代码而言，最终会走到createElm中。createElm(vnode, [], body, textnode)。
 在createElm中，对vnode进行处理，最终通过调用nodeOpts中的各种操作，完成对dom节点的增删改查操作。同时，通过调用对应的钩子函数，完成对dom属性的变更操作。
 
 在createElement之后，如果当前的vnode有parent，即当前vnode是一个渲染vnode，而vnode.parent是一个组件占位符vnode。更新vnode.parent所绑定的elm为创建新节点后的vnode的elm。同时调用相关的钩子函数完成占位符节点的更新操作。
 
 之后，移除oldVnode，增加vnode到parentElm。将子组件由子到父调用mounted钩子函数。
 
 至此patch的初始创建过程完成。最后返回新创建或者更改后的最新的vnode.elm即真实的dom节点。
 
 -----------------------
 
 下面来看下createElm的代码。createElm主要做了以下几件事：
 要说的都在注释里了，就不再额外阐述了。总之createElm就做了创建并插入dom节点这件事。
 
 ``` javascript
 // src/core/vdom/patch.js
 function createElm (
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
  ) {
    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // This vnode was used in a previous render!
      // now it's used as a new node, overwriting its elm would cause
      // potential patch errors down the road when it's used as an insertion
      // reference node. Instead, we clone the node on-demand before creating
      // associated DOM element for it.
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    vnode.isRootInsert = !nested // for transition enter check
    // 1. 尝试去创建组件，如果当前vnode是占位符vnode，则会返回true，
    //    也就不走之后的流程了。示例代码中第一次到达此处的是一个组件的vnode，
    //    它是占位符vnode，所以会进入createComponent执行创建VueComponent实例。
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
      return
    }

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    // 如果tag存在，此处的tag应为一个dom节点的tagname。组件的tag会在前边createComponent被处理掉。
    if (isDef(tag)) {
      if (process.env.NODE_ENV !== 'production') {
        if (data && data.pre) {
          creatingElmInVPre++
        }
      }
     
      2. // 获取到vnode.tag，创建真实的dom节点
      vnode.elm = vnode.ns
        ? nodeOps.createElementNS(vnode.ns, tag)
        : nodeOps.createElement(tag, vnode)
      setScope(vnode)

        // 此处移除了weex里的逻辑，只留下web平台的代码
        // 3. 调用createChildren，对vnode的children进行深度遍历，直到
        createChildren(vnode, children, insertedVnodeQueue)
        if (isDef(data)) {
          // 调用对应的钩子函数
          invokeCreateHooks(vnode, insertedVnodeQueue)
        }
        // 4. 将当前vnode.elm插入到paremtElm中。完成dom创建操作
        insert(parentElm, vnode.elm, refElm)

      if (process.env.NODE_ENV !== 'production' && data && data.pre) {
        creatingElmInVPre--
      }
    } else if (isTrue(vnode.isComment)) {
      // 如果是注释vnode，直接创建注释节点并插入
      vnode.elm = nodeOps.createComment(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    } else {
      // 否则，把当前vnode视为文本节点，创建文本节点并插入
      vnode.elm = nodeOps.createTextNode(vnode.text)
      insert(parentElm, vnode.elm, refElm)
    }
  }
 ```
 
 --------------------------------
 
 既然createElm的主要逻辑在createComponent中，那就看下createComponent吧。在createComponent中，根据当前的组件vnode，会生成VueComponent实例并执行相应的初始化过程，最后调用子组件实例的$mount方法。进行子组件相关dom节点的创建工作。
 
 ``` javascript
 // src/core/vdom/patch.js
 function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        // 1. 调用组件vnode.data.hook上的init方法，执行了子组件构造函数的_init函数，并且执行了子组件的$mount方法。
        i(vnode, false /* hydrating */)
      }
      
      // 在调用init钩子后，如果当前vnode是一个子组件，它应该已经创建
      // 了一个组件实例并且挂载了它(即调用了$mount方法。
      // 子组件也已经设置了elm属性，elm是子组件生成的对应的dom节点。
      if (isDef(vnode.componentInstance)) {
        // componentInstance就是子组件对应的vm实例。
        
        // 清空子组件vnode.data.pendingInsert，并把它存在insertedVnodeQueue上。
        // 设置当前vnode.elm属性为componentInstance的$el。
        initComponent(vnode, insertedVnodeQueue)
        // 将vnode对应的dom节点插入到父节点指定的位置。
        insert(parentElm, vnode.elm, refElm)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }
  }
 ```
 
 在生成子组件vm实例时，会调用vm实例上的\$mount方法。而$mount方法又要走到patch function中。最终也会进入到createElm方法中。不过这时调用createComponent会返回false。因为此时的vnode.tag为App.vue中的父级div()。
 
 之后进入createElm的第二步、第三步、第四步。这个过程就不再赘述了。反正就是一个createElm和createChildren互相调用的过程。在这个过程中，创建了对应的dom节点，并通过调用invokeCreateHook函数完成对dom属性的操作。
 
 最后经过createElm，已经生成了一个属性完备的dom节点，并且已经插入到了body中。初次的vnode渲染dom就完成了。
 
 -------------------------------------------
 
 ### entry-runtime.js和entry-runtime-with-compiler.js的区别及适用场景 
 
 从字面意思看，两个文件就差了compiler即编译器。翻阅代码可得，entry-runtime-with-compiler版本是对entry-runtime.js版本的扩展，即重写了entry-runtime.js中Vue的原型方法$mount。
 
在vue-cli初始化项目时，会让我们选择使用哪个版本的vue，官方推荐的是选择with-compiler版本的。

其实，with-compiler版本比runtime版本多的就是一个编译器，就是在初始化vue时的render函数。如果没有选择这个版本，我们需要在初始化vue实例时手动传入render函数配置项。而不能直接以sfc(单文件组件)的方式来写vue。

而在手动传入render函数时，我们要以创建虚拟dom的形式传入参数。例如：
```
new Vue({
    render(h) {
        return h('div', { staticClass: 'app', data: {} }, 'hello,world');
    }
})
```

而使用带compiler版本的，我们可以这么写:

``` javascript
<div id='app'>hello,world</div>

new Vue({
    el: '#app'    
})
```

------------------------------

这期就分析到这里了。拜拜~