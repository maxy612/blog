---
title: Vue源码分析——computed和watch的实现及执行过程分析
date: 2019-06-24 23:23:31
tags: ['Vue']
summary: 之前对vue就挺有兴趣的，工作中也是以vue为技术栈的，工作之余对vue源码进行了部分研究。
---

接触Vue也有两年了，在工作中主要的技术栈也是vue。今天来聊下vue中computed和watch是怎么实现的。其中会涉及到vue的响应式系统，会在另一篇文章中介绍。
    
### computed和watch的用法

在使用vue时，我们会用到vue的众多配置项，其中就包括computed和watch。例如：
``` javascript
<div id='app'>
    <p>{{ computedA }}</p>
</div>

<script>
new Vue({
    el: '#app',
    data() {
        a: 1,
        b: 2,
        c: {
            d: 3
        }
    },
    
    computed: {
        computedA() {
            return this.a + 30;
        }
    },
    
    watch: {
        c: {
            deep: true,
            handler(newVal, oldVal) {
                console.log('newVal: ', newVal);
            }
        },
        
        b(newVal, oldVal) {
            console.log('newVal: ', newVal);
        }
    },
})   
</script>
```

当data中的a发生变化时，computedA也会重新计算获得最新的值。
当data中的c或b发生变化时，watch中对应的函数也会执行。
    
### computed和watch的实现
翻开vue的源码，我们分析下computed和watch的实现。
Vue做为一个构造函数，首先会通过调用原型方法_init执行实例化的过程。而对于computed和watch的处理，就在_init函数中。
在init中，执行了一系列对传入的options初始化的过程。其中，执行了initState的函数。在initState函数中，有对computed和watch的处理。不仅如此，还包含了对data，methods，props的处理
``` javascript
if (opts.computed) initComputed(vm, opts.computed)
if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
}
```
其实对于computed和watch的处理，差别不大。最后都生成了watcher实例。当相应的数据发生变化时，通知对应的watcher实例执行更新。

#### 下面来分析下computed的处理过程(服务端渲染暂不做分析)。
首先，大致说下处理流程。

initComputed -> computed watcher(vm, get, {lazy: true}) -> defineComputed(vm, key, userDef)

其中，处理computed的核心就在于initComputed中。在initComputed中，大致执行了以下四个过程：

1. 在vm上挂载_computedWatchers。
2. 遍历computed，从userdef中获取getter
3. 生成computed watcher(vm, getter, noop, { lazy: true })
4. 定义vm[key]的get和set。通过执行defineComputed(vm, key, userDef)实现 // 在子实例上会提前执行。

``` javascript
function initComputed (vm: Component, computed: Object) {
  // 第一步 在vm上挂载_computedWatchers
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  // 第二步 遍历computed，从userdef中获取getter
  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      // 第三步 生成computed watcher(vm, getter, noop, { lazy: true })
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      // 第四步 定义vm[key]的get和set
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```
    
下面来分析下生成computed watcher的过程。

根据传入的参数vm, getter, noop, ，我们可以大致得到这样一个watcher实例：
<pre>
{
    vm: vm,
    getter: getter,
    cb: noop,
    lazy: true,
    dirty: true,
    ...reset
}
</pre>

在watcher实例初始化的过程中，会对实例挂载一系列的属性。由于this.lazy是true，所以初始化时得到的 this.value = undefined。
``` javascript
// 来自 watcher.js
this.value = this.lazy
      ? undefined
      : this.get()
```

这样computed的初始化就完成了。

假设html模板中用到了computed中的值。在Vue实例init的最后，会调用vm.$mount方法，进而执行到mountComponent函数，mountComponent会生成一个渲染watcher实例。

当render watcher执行时，会访问到计算属性的值，进而会触发计算属性的get。计算属性的get在defineComputed中已经定义过了，如下：
``` javascript
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

然后会执行computedGetter。此时，watcher存在，且watcher.dirty为true。会执行watcher.evaluate获得计算属性的值。而在evaluate中，会调用watcher.get方法。
``` javascript
  // 选自 watcher.js
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }
```
在执行watcher.get方法时，会将Dep.target设置成当前的watcher, 并调用计算属性的函数。即实例中computedA所对应的函数。在执行这个函数时，又会访问到data中a的值，也即是vm[a]的值。而在vue的响应式系统中，在初始化时对data中的数据进行了响应式观测，并定义了get和set函数，生成了对应的依赖收集器dep。

此时, 计算属性在计算时访问到了vm[a]，触发了vm[a]对应的dep进行依赖收集。
``` javascript
    // 来自 observer/index.js defineReactive
    const value = getter ? getter.call(obj) : val
    // Dep.target此时为computed watcher。
    if (Dep.target) {
        // 使得computed watcher和dep互相添加对方
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
    return value
```

这样，vm[a]的dep就有了computed watcher。当vm[a]发生变动时，就能够通知到computed watcher。

当computed watcher的evaluate执行完后，Dep.target会变成render watcher。dirty会被置为false。接着往下进行，会执行到watcher.depend(watcher是computed watcher)。
``` javascript
  // 来自 watcher.js
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
```
此时，depend函数会把刚才computed watcher添加的deps也添加到render watcher中去，使得vm[a]的dep和render watcher也能互相添加到彼此。这样，当vm[a]发生变化时，render watcher也会执行。

当watcher.depend执行完后，返回watcher.value。此时，computed watcher就完成了一次计算过程。

当vm[a]发生变化时，会触发其dep的set。进而触发dep.notify。此时，dep.subs中有两个watcher实例。一个是computed watcher，另一个是render watcher。首先执行computed watcher的update，然后执行render watcher的update。
``` javascript
  // from watcher.js
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      // computed watcher会走到这里
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      // render watcher会走到这里
      queueWatcher(this)
    }
  }
```
computed watcher的update只是简单地把dirty置为了true，以便在下次访问到计算属性时，通过执行evaluate来计算最新的值。

render watcher会被放入执行队列中。将当前主线程代码执行完毕后，在下一次事件循环中执行。

最终，随着render watcher.get的执行，计算属性会被重新获取，进而触发计算属性的getter，然后通过执行computed watcher的evaluate重新获取计算属性的最新值并返回。

以上就是computed的大致执行过程和数据更新过程。

-----------------------------------

下面来分析下watch的执行过程。

从initWatch开始讲起。Vue -> _init -> initState -> initWatch

在initWatch中，主要做了下面几件事：
1.  遍历watch配置项，调用createWatcher(vm, key, val);
2.  在createWatcher中对watcher的参数进行格式化，统一格式，然后调用Vue原型方法$watch。
3.  在$watch中，创建user watcher(vm, expOrFn, cb, { user: true, immediate })。如果配置项中有immediate且为true，在user watcher初始化后，立即调用一次val，即观察数据发生变化后配置的回调函数。之后，返回一个销毁该watcher的函数。

watch最终会创建user watcher。下面我们用一个例子来分析下：
``` javascript
new Vue({
    el: '#app',
    data() {
        return {
            a: 1,
            b: {
                c: 2,
                d: {
                    e: 5
                }
            }
        }
    },
    
    computed: {
        computedA() {
            return this.a + 10;
        },        
    },
    
    watch: {
        computedA(newVal, oldVal) {
            console.log(newVal);
        },
        
        a(newVal, oldVal) {
            console.log(newVal);
        },
        
        b: {
            deep: true,
            handler(newVal) {
                console.log(newVal);
            }
        }
    }
})
```

三个watch对应了三种不同的观察对象。

#### 下面先介绍一般情况，即watch中对a的观测。

- 首先，在initWatcher时，通过for in 遍历，会直接进入到createWatcher中；
- 然后，在createWatcher中，由于handle就是普通的function，所以会直接进入vm.$watch中。
- 之后，在$watch中，创建user watcher(vm, 'a', fn, { user: true });
- 最后，会进入user watcher实例初始化的过程。最终，生成了一个数据a的user watcher。大致如下：
<pre>
{
    vm: vm,
    getter: fn, // 由parsePath生成的
    cb: noop,
    lazy: false,
    user: true,
    ...reset
}
</pre>

在user watcher生成最后，会调用watcher.get方法。
``` javascript
    // from watcher.js constructor
    this.value = this.lazy
      ? undefined
      : this.get()
```

进入watcher.get方法后，会将当前的Dep.target设置为当前user watcher。

此时，调用watcher.getter，会访问到vm['a']，而vm['a']在之前的initData中被observe并且设置了getter和setter，此时会触发vm['a']的getter。vm['a']的dep和当前的user watcher会通过dep.depend互相添加，vm['a']的dep的subs中有该user watcher，该user watcher的newDeps中也会有vm['a']的dep。

当vm['a']发生变化时，会触发vm['a']的setter，进而vm['a']的deps调用notify方法，然后调用之前添加user watcher的update方法，被添加到queueWatcher中。

最终，随着任务队列的执行，会执行到user watcher的run方法。之后会调用watcher.get重新求值。
``` javascript
    // from watcher.js run
    
    // 当最新计算的值和watcher.value不等或者value是object或者watcher.deep为true时，会执行以下代码
    const oldValue = this.value
    this.value = value
    if (this.user) {
        // user watcher 会走到这里，因为 watcher.user = true
        try {
            this.cb.call(this.vm, value, oldValue) // 调用观察数据的回调函数
        } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
    } else {
        this.cb.call(this.vm, value, oldValue)
    }
```

观察普通值的初始化及数据变动分析到这里就结束了。

#### 下面我们来分析下对computedA的观察。在这里，和普通值观测相同的相同的地方就不细说了，重点说不同的地方。

不同之处在于生成的user watcher的不同。其实是又回到了计算属性的get过程。

生成user watcher之后，调用watcher.get。此时，会访问到vm['computedA']，而vm['computedA']是一个计算属性，在watch之前已经初始化了一个computed watcher。此时的Dep.target是user watcher。在访问vm['computedA']时，代码会执行到这个地方：
``` javascript
// from state.js 
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    // 代码会执行到这里
    if (watcher) {
      if (watcher.dirty) {
        // 通过对computed watcher的重新求值，使得vm['a']的dep中有该computed watcher，该computed watcher的deps中也有vm['a']的dep。
        watcher.evaluate()
      }
      if (Dep.target) {
        // computed watcher重新求值后，Dep.target会重新退回到user watcher。通过watcher的depend，使得vm['a']的dep和user watcher也互相添加。
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

当数据vm['a']发生变动时，此时vm['a']的deps中既有computed watcher，也有user watcher。其中computed watcher在前。此时，会分别执行两个watcher的update方法。

computed watcher的update之前已经说过，不再赘述。
之后，user watcher会被queueWatcher添加到执行队列。最终执行watcher.run方法。在watcher.run中，访问到了vm['computedA']，此时，重新触发了对vm['computedA']的getter。vmp['computedA']会再次调用watcher.evaluate方法完成求值并返回。此时能够获取到最新的computedA的值，然后执行和观察vm['a']一样的过程，不再详述。

#### 下面说下对b的观察。

和对a的观察相比，主要的不同主要集中于初始化的createWatcher和watcher.get方法中。除此以外，和观察a的区别不大。

1. createWatcher中，handle为object，因此会进入到if分支
``` javascript
// from state.js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    // 会进入到这里，对传入的handle进行处理
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```
2. 在watcher.get中，有对watcher.deep的处理，即对深度观察的处理。
``` javascript
  get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      // 会进入到下边的if分支。
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
```

通过traverse，使得vm['b']下的每一层数据都被访问到了，其对应的dep和当前的b的user watcher都相互添加。最终实现 当vm['b']下任意一层数据变更，都能够通过数据的dep触发notify，进而调用到该user watcher的update方法，从而执行配置的观察回调函数。

-------------------------------

对watch和computed的讲解分析就先到这里了。能够这样理一遍，也算是有所收获。如果有错误，欢迎指出。谢谢。