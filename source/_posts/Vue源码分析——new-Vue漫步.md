---
title: Vue源码分析——new Vue漫步
date: 2019-07-09 00:30:12
tags: ['Vue']
summary: 对new Vue的实例化过程进行简要分析。
---


我们在使用Vue开发项目时，new Vue({ el: '#app', methods: {}, ...rest })。那么在这个过程中，发生了什么呢？
以web平台为例，vue构造函数有外到内引用顺序依次为：

1. platforms/web/runtime/index.js(设置了Vue.config上的一些属性，挂载__patch__, $mount到vue的原型上)
2. core/index.js(为vue挂载一些静态方法即全局api，Vue.extend, Vue.component等。定义\$isServer, $ssrContext到vue的原型上)
3. core/instance/index.js(定义Vue构造函数，依次调用initMixin, stateMixin, eventsMixin, lifecycleMixin, renderMixin）

------------------------------------------

接下来我们简要分析一下new Vue都做了什么操作。

在vue的构造函数中，只调用了原型上的_init方法，并传入options。在_init方法中，开始了vue实例化的过程。主要做了以下几件事：

1. 合并配置
2. 代理vm
3. 初始化vm关于生命周期，父子关系，事件处理以及渲染相关的属性
4. 调用钩子函数beforeCreate
5. 处理options中的inject和provide配置项
6. 处理options中的props, methods, data, computed, watch配置项。
7. 调用钩子函数created
8. 如果options.el存在，调用vm.\$mount，$mount来自于Vue.prototype。

--------------------------------------------

下面着重说一些第一步和第六步。第七步会单独作为一个章节来讲。

#### 合并配置
``` javascript
vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
)
    
function resolveConstructorOptions (Ctor: Class<Component>) {
  // 乍一看代码很多，其实new vue时并没有走if分支，获取到Vue.options并返回。
  let options = Ctor.options
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    if (superOptions !== cachedSuperOptions) {
      // super option changed,
      // need to resolve new options.
      Ctor.superOptions = superOptions
      // check if there are any late-modified/attached options (#4976)
      const modifiedOptions = resolveModifiedOptions(Ctor)
      // update base extend options
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
```

mergeOptions合并就是在合并Vue.options和传入的options。下面来看一下mergeOptions:

``` javascript
function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }
  
  //  格式化child的props配置项为统一的格式：props: { propsA: { type: xx }, propsB: { type: xx }}
  normalizeProps(child, vm)
  
  // 格式化child的inject配置项为统一格式：inject: { injectA: { from: xx } }
  normalizeInject(child, vm)
  
  // 格式化child的directives为统一格式： directives: { bind: fn, updaet: fn }
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  // 只有合并过的options有_base属性。
  if (!child._base) {
    if (child.extends) {
      // 处理options.extends的合并
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      // 处理child.mixins的合并
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  
  // 选择对应的合并策略。不同的key有不同的合并策略，如data, components, 
  // filters等。合并策略取自config.optionMergeStrategies中，用户可以自定义合并策略。否则，采用默认的合并策略。
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```

执行完合并策略后，将合并后的options赋值给vm.$options以供后续使用。

------------------------------------

#### 处理配置项data, props, methods, computed, watch

对于props. 调用initProps(vm, opts.props)。

1. 在initProps中，遍历options.props，将props的keys进行收集，放在vm.$options._propKeys上。
2. 通过defineReactive对props中的值定义在vm._props上并设置成响应式属性，便于当props发生变化时触发对应的watcher更新视图。
3. 将对vm[key]的访问代理到vm._props[key]上。


对于methods，调用initMethods(vm, opts.methods)

1. 对methods的key和value进行检测。对于key，不能是props中的key，不能是vm上的以$或者_开头的属性或方法。对于value，必须为function
2. vm[key] = value.bind(vm)


对于data，调用initData(vm)。

1. 获取到data中的值
2. 检查data中的key是否和methods或者props中的有重复，如果重复，给出警告
3. 将对vm[key]的访问代理到vm._data[key]上
4. 对获取的data进行遍历并定义为响应式属性，便于之后收集依赖watcher。


对于computed和watch的处理，可在[Vue源码分析——computed和watch的实现及执行过程分析](https://maxy612.cn/2019/06/24/Vue%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E2%80%94%E2%80%94computed%E5%92%8Cwatch%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8F%8A%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/)查看。

----------------------------------

以上就是对Vue过程的大致分析，之后会针对某个过程进行单独分析。
