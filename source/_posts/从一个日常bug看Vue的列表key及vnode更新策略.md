---
title: 从一个日常bug看Vue的列表key及vnode更新策略
date: 2019-08-16 22:42:56
tags: ["Vue", "Bug"]
summary: 之前在做h5活动的时候，遇到了一个关于vue中列表渲染的bug。当然，bug是我自己写的，和vue没有半毛钱关系。不过在解决bug的过程中，对vue的patch diff的过程进行了一番研究。
---

之前在做h5活动的时候，遇到了一个关于vue中列表渲染的bug。当然，bug是我自己写的，和vue没有半毛钱关系。不过在解决bug的过程中，对vue的patch diff的过程进行了一番研究。

在探究过程中，涉及到了vue列表渲染的key的研究，以及vue渲染函数及生命周期的执行过程分析。

### bug的由来及重现
场景是这样的：
    1. 用vue的v-for做列表渲染。列表中有图片和文字。
    2. 点击按钮，会往列表数据的最前面增加一条数据。
    3. 图片为了做onerror的处理，我自己封装了一个image的组件。
    
然而，在点击按钮后，数据发生了变化，但是图片显示却发生了错位，即首项的图片并没有正确更新，而是直接显示的数据变化前的第一条数据的图片。demo如下：

```javascript
// App.vue
<div id="app">
    <div class="list">
      <!-- 就是渲染了一个普通的列表 -->
      <div v-for="(item, index) in list" :key="index">
        <mt-image :src="item.logoUrl" />
        <p class="desc">
          <span class="nickname">{{item.nickName}}</span>
          <span class="detail">{{item.desc}}</span>
        </p>
      </div>
    </div>
    
    <button @click="loadImg">addData</button>
</div>

<script>
// 图片组件
import mtImage from "./components/image";

export default {
  name: "App",
  data() {
    return {
      list: [
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"码"卡',
          cardType: 2,
          btnType: 0
        },
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"码"卡',
          cardType: 2,
          btnType: 0
        },
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"洋"卡',
          cardType: 1,
          btnType: 0
        },
      ]
    };
  },

  methods: {
    loadImg() {
      const addMsg = {
        nickName: 'maxy612',
        logoUrl: 'https://user-gold-cdn.xitu.io/2019/8/15/16c95b88c07c1f83?w=132&h=132&f=jpeg&s=4850',
        desc: '测试添加数据'
      };

      this.list.unshift(addMsg);
    }
  },

  components: { mtImage }
};
</script>

// image.vue
// 虽然这个组件的意义不大，但是当时大概就是这么想的。
// 其实看到这里对vue稍微熟悉点的就能看出问题了。
<template>
    <img :src="relSrc" alt="">
</template>

<script>
import defaultImg from '../assets/logo.png';

export default {
    name: 'mt-image',
    props: {
        src: {
            type: String,
        }
    },

    data() {
        return {
            relSrc: defaultImg,
        };
    },

    mounted() {
        this.loadImage();
    },

    methods: {
        loadImage() {
            const img = new Image();
            
            img.src = this.src;
            img.onload = () => {
                this.relSrc = this.src;
            }
        }
    }
}
```

上述代码，当点击按钮增加数据之前，是这么显示的：


![数据改变前的展示效果](https://user-gold-cdn.xitu.io/2019/8/15/16c95b5eb7e9fef4?w=374&h=674&f=jpeg&s=33304)

而点击了addData按钮之后，会在列表的最前面插入一条测试数据（详见loadImg函数），此时显示结果是这样的：


![改变后的数据展示效果](https://user-gold-cdn.xitu.io/2019/8/15/16c95b76b1448903?w=305&h=887&f=jpeg&s=39292)

但是，增加的那条addMsg的logoUrl是这样的：

![增加的数据中的logourl](https://user-gold-cdn.xitu.io/2019/8/15/16c95b88c07c1f83?w=132&h=132&f=jpeg&s=4850)

按正常展示(或者说我们想让展示的结果)来说，改变后数据的第一个图片上面的"狮子头"图片，然而显示展示的还是我自己的头像...

------------------------------
### bug分析

首先来分析下执行过程：
1. 增加该组件的渲染watcher到data中的list。

    首先，当vue通过$mount进行渲染时，此时生成了渲染watcher。而渲染watcher在执行时，访问到了data中的list。此时，触发了list的getter函数，该渲染watcher被添加到了list的依赖收集器dep中。当list变化时，触发其dep.notify方法，进而执行到渲染watcher的update方法，也就是vm._update(vm._render(), hytrating)函数，该组件会进行重新渲染。

2. list变化触发组件执行渲染watcher的update方法，进行重新渲染。在渲染过程中，会经过vm._update(vm._render(), hydrating) -> vm._update -> vm.__update__(preVnode, vnode) -> patch -> patch -> patchVnode。
``` javascript
function createPatchFunction(backend) {
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
          // 会进入到这里
        // 真正的update，新旧vnode对比更新
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
          // 略去
      }
}
```

3. 在patchVnode的过程中，进行patch diff，完成dom节点和vnode的比较更新过程。而这一步，就是问题的所在了。首先我们先看下这时候的oldvnode和vnode都是什么（仅看和列表相关的）。


![旧的列表vnode](https://user-gold-cdn.xitu.io/2019/8/16/16c960d3837dfc05?w=832&h=535&f=jpeg&s=87418)


![新的列表vnode](https://user-gold-cdn.xitu.io/2019/8/16/16c960ded7080799?w=872&h=452&f=jpeg&s=69853)

> 从vnode的对比结果可以看到，新的vnode已经多了一个children。此时看一下list的第一项是什么。

![list的第一条数据](https://user-gold-cdn.xitu.io/2019/8/16/16c961109200bd07?w=976&h=100&f=jpeg&s=33052)

> 可以看到，数据已经是我们添加过后的数据了。


接下来我们再来看看变化过后的列表对应的dom。

![最新的dom](https://user-gold-cdn.xitu.io/2019/8/16/16c9614c1ace6bd8?w=1038&h=143&f=jpeg&s=85430)

**从vnode和list以及dom来看，数据已经更新了，但是并没有真正应用到dom节点上。**

那么问题就在patchVnode上了。对于patchvnode的过程，单纯的文字解释也很难懂，在这里推荐阅读黄轶大佬的解读文章[vue组件更新](https://ustbhuangyi.github.io/vue-analysis/reactive/component-update.html#%E6%96%B0%E6%97%A7%E8%8A%82%E7%82%B9%E4%B8%8D%E5%90%8C)。

**在Vue文档中，当列表渲染时，官方推荐我们为列表中的项指定一个唯一的key值。这个key值用于在patchVnode时作为判断sameVnode的重要依据。当key值相同且满足其它相关条件(在代码中会解释)时，新旧vnode便可以判定为sameVnode。这也就意味着旧的vnode所对应的dom节点可以被重用。然后把符合sameVnode新旧vnode再次进行patchvnode，在patchvnode中完成相关dom节点属性的更新，从而实现了vnode到真实dom的改动。**

下面我们来分析下具体的执行过程。
```javascript
// 判断samevnode，除了key相同，还要求两个vnode的tag, isComment, inputType相同并且data同为有定义或无定义；对于异步占位符vnode，暂时先不做分析。
function sameVnode() {
    return (
        a.key === b.key && (
        (
            a.tag === b.tag &&
            a.isComment === b.isComment &&
            isDef(a.data) === isDef(b.data) &&
            sameInputType(a, b)
        ) || (
            isTrue(a.isAsyncPlaceholder) &&
            a.asyncFactory === b.asyncFactory &&
            isUndef(b.asyncFactory.error)
        )
    )
  )
}

function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    // 将新生成的vnode的elm指向之前的oldvnode的elm（即之前vnode所对应的dom节点，便于复用，而不用创建新的节点）
    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    // 进行dom属性的更新操作
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      // 如果不是文本节点
      if (isDef(oldCh) && isDef(ch)) {
        // 如果新旧vnode的children都存在，并且不相等，进入updateChildren的过程，这个过程中会对新旧children进行头头比较、尾尾比较，头尾比较及尾头比较，以及根据key在旧的vnode的children中寻找和新vnode的children的起始元素key一致的属性。
        // 在比较过程中，通过判断
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        // 如果旧的vnode的children不存在，新的vnode的children存在，直接添加children对应的节点到vnode对应的dom中
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        // 如果新vnode的children不存在，旧的vnode的children存在，直接简单粗暴的移除vnode的elm所对应的children子dom节点。
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        // 如果新旧vnode的children都不存在，则直接设置vnode对应的dom节点elm的textcontent为空
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      // 如果新旧vnode的文本不同，直接新的替代旧的
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

对于updateChildren的过程，先大致说下它的更新过程：

```javascript
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    // 1. 定义oldch和newch的开始和结束位以及各自对应的vnode。
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    // 是在vm._update时带过来的，默认为false或undefined,所以canMove为true
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        // 如果oldch的起始vnode为空，将起始元素指向后一个vnode
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        // 如果oldch的末尾vnode为空，将末尾元素指向前一个vnode
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // newch和oldch进行起始元素的比较，如果key相同及其它条件相同，则复用之前vnode对应的dom节点，调用patchVnode，进行dom属性的更新。然后newch和oldch的起始元素分别指向下一个vnode。
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // newch和oldch进行末尾vnode的比较，如果是samevnode，则复用vnode对应的dom，然后通过patchvnode进行dom属性的更新及子children的比较。然后newch和oldch的末尾vnode分别指向前一个vnode。
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        // 如果oldch的起始vnode和newch的末尾vnode是samevnode，调用patchvnode进行dom属性更新及子children的比较。然后将oldch的起始vnode移到oldch的末尾vnode之后。最后将oldch的起始vnode指向后一个vnode，将newch的末尾vnode指向前一个vnode。
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        // 同上分析过程。不过是oldch的末尾vnode和newch的起始vnode进行比较。
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        // 如果在进行了头头、头尾、尾头、尾尾比较之后仍然没有找到samevnode，则将oldch的起始vnode到末尾vnode的key进行提取，
        // 形成一个{key: vnode}的map。然后取出newch的起始vnode的key,在map中查找看有无匹配到的。如果没有匹配到，
        // 则直接创建元素。如果匹配到了，并且匹配到的vnode和newch的起始vnode是samevnode,则进行patchvnode更新dom，
        // 并且将oldch的匹配到的vnode对应的dom移到oldch的起始vnode的前面。如果不满足samevnode，
        // 则直接添加newch的起始vnode对应的dom到父节点中。
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    
    // 如果oldch已经遍历完了，那就把newch的起始元素到末尾元素都添加到父节点中。
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      // 如果newch已经遍历完了，那就把oldch的起始元素到末尾元素都从父节点中移除。
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

updateChildren的过程大致分析完了，下面回到我们这个bug上：

**当数据增加之后，旧列表vnode的children为3个，新列表vnode的children为4个。然后进行updateChildren。**

**首先进行新旧vnode的children的起始元素比较。这个时候，由于key是index，而起始元素的index都为0，而且其它判断samevnode的条件也符合，因此进入到patchvnode的过程中。**

**这个时候，新旧列表的第一个vnode的数据发生了变化。新的logoUrl，nickname，desc被更新到了对应的dom节点上。然而，这时候，mt-image组件收到了一个新的src属性值，但是它在内部对新传递过来的src没有做任何处理,也没有任何的观察者(watcher)收集了src。而对props中src的处理，只在mt-image组件创建并执行mounted之后才会进行，在这里这个mt-image组件被复用了，因此并不会执行对src的处理，导致内部img标签上的src依然为之前的src，所以此时就出现了图片展示异常问题。**

-------------------------------

### bug的解决方案
原因找到了，如何修复呢？在这里，我一共总结了三种解决方案：

**1. 在mt-image组件中对props的src进行观察,即在watch中观察src，当src发生变化后，执行loadImage函数。**
```javascript
// image.vue
<template>
    <img :src="relSrc" alt="">
</template>

<script>
import defaultImg from '../assets/logo.png';

export default {
    name: 'mt-image',
    props: {
        src: {
            type: String,
        }
    },

    data() {
        return {
            relSrc: defaultImg,
        };
    },

    mounted() {
        this.loadImage();
    },

    methods: {
        loadImage() {
            const img = new Image();
            
            img.src = this.src;
            img.onload = () => {
                this.relSrc = this.src;
            }
        }
    },

    watch: {
        src: 'loadImage'
    },
}
</script>  

// img的src属性更新的调用关系图
props src: changed -> props src: set -> props src: dep.notify -> user-watcher src -> this.loadImage -> 
-> this.relSrc = xxx -> data relSrc: set -> data relSrc: dep.notify -> vm._update(vm._render(), hydrating)
```
该方法实质上是在组件内部生成了一个user watcher。在mt-image初始化的时候，会对watch中的配置项生成相应的watcher。

下面我们来理一下src的watcher实例、data中relSrc和mt-image组件的渲染watcher以及props中的src之间的关系。
    
1. 当mt-image在执行vm._update(vm._render(), hydrating)的过程中，会访问到data中的relSrc, 进而触发了relSrc的依赖收集。relSrc的依赖收集器dep将渲染watcher加入到它的subs中。当relSrc变化时，触发relSrc的set，进而调用其依赖收集器的notify方法，触发组件渲染watcher的重新执行。
    
2. 接下来说说props中src的变化如何引起img的src属性变更，分为两步：
        
    前提条件：在mt-image组件实例化时，组件会执行_init方法，接着会调用到initState，进而调用initProps和initWatch方法。initProps通过defineReactive对props中的src做数据劫持，initWatch方法会遍历
    组件配置的watch中的每一项，并生成对应的user watcher。
        
    * 在生成src的user watcher时，会触发对props中src的访问，进而该user wather被添加到props中src的依赖收集器中。当src发生变化时，会触发该user watcher的update方法，进而执行配置的回调函数。
        
    * 在执行函数时，当图片加载成功后会进入onload中，此时会对relSrc重新赋值，进而触发relSrc的set, 从而调用之前第一步中添加的渲染watcher进行dom节点的更新操作。
        


**2. 去掉mt-image组件，直接用img标签代替**

``` javascript
// app.vue
<template>
  <div id="app">
    <div class="list">
      <div v-for="(item, index) in list" :key="index">
        <!-- 改动在这里哦 -->
        <img :src="item.logoUrl" :onerror="defaultImg" />
        <!-- <mt-image :src="item.logoUrl" /> -->
        <p class="desc">
          <span class="nickname">{{item.nickName}}</span>
          <span class="detail">{{item.desc}}</span>
        </p>
      </div>
    </div>

    <button @click="loadImg">addData</button>
  </div>
</template>

<script>
import mtImage from "./components/image";
import defaultImg from './assets/logo.png';

export default {
  name: "App",
  data() {
    return {
      defaultImg: `this.src="${defaultImg}"`,
      list: [
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/13",
          desc: '抽得一张"码"卡',
          cardType: 2,
          btnType: 0
        },
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"码"卡',
          cardType: 2,
          btnType: 0
        },
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"洋"卡',
          cardType: 1,
          btnType: 0
        },
      ]
    };
  },

  methods: {
    loadImg() {
      const addMsg = {
        nickName: 'maxy612',
        logoUrl: 'http://thirdwx.qlogo.cn/mmopen/vi_32/AELVSluys1wCA8zzSqJicCxPhHNdSSvYWW3Rlp6jFh5WlNVeeWqVBVmQV8p9KibApfKaYbGQbib8Mpdxh2YK0Ulibw/132',
        desc: '测试添加数据'
      };

      this.list.unshift(addMsg);
    }
  },

  components: { mtImage }
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

button {
  width: 100px;
  line-height: 30px;
  text-align: center;
}
</style>
```

这种方法的处理原理就是让img回到正常的更新流程中，和其同级的span一起在patchVnode中被更新。具体的更新操作发生在patchVnode中执行cbs.update时。在这里就不做过多介绍了。

**3. 给list增加一个列表项唯一的id值，列表循环时key为唯一的id值**
```javascript
// app.vue 只列出改动点, mt-image无变化
// 用id代替index
<div v-for="item in list" :key="item.id">
    <mt-image :src="item.logoUrl" />
    <p class="desc">
        <span class="nickname">{{item.nickName}}</span>
        <span class="detail">{{item.desc}}</span>
    </p>
</div>

<script>
let n = 4;

export default {
  name: "App",
  data() {
    return {
      list: [
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/13",
          desc: '抽得一张"码"卡',
          cardType: 2,
          btnType: 0,
          id: 1,
        },
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"码"卡',
          cardType: 2,
          btnType: 0,
          id: 2,
        },
        {
          nickName: "马晓阳",
          logoUrl:
            "http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8yxh4HpjYVZObxCV9yIteHCcg8XtThmY2iahwWQAXHcnfP9x0iah91tLITOn9FZIfbxnHE5QdicVtQ/132",
          desc: '抽得一张"洋"卡',
          cardType: 1,
          btnType: 0,
          id: 3
        },
      ]
    };
  },

  methods: {
    loadImg() {
      const addMsg = {
        nickName: 'maxy612',
        logoUrl: 'http://thirdwx.qlogo.cn/mmopen/vi_32/AELVSluys1wCA8zzSqJicCxPhHNdSSvYWW3Rlp6jFh5WlNVeeWqVBVmQV8p9KibApfKaYbGQbib8Mpdxh2YK0Ulibw/132',
        desc: '测试添加数据',
        id: n++
      };

      this.list.unshift(addMsg);
    }
  },

  components: { mtImage }
};
```
这种方法的原理可以从patchVnode和updateChildren中找到答案。

**更换了key值为列表项唯一id时，就大不一样了。当新增msg到列表项最前面后，在接下来的updateChildren时，进行新旧children的第一项对比。而新vnode的children的第一项的id为4，在updateChildren的匹配过程中，未匹配到任何能复用的节点，于是这时候新增加的列表数据项就会被当作新节点创建，之后再进行后续的操作.**

----------------------

由于codesandbox最近打开比较慢，就暂时不提供线上demo了，全部代码在上边都有贴出来。

那今天的分析就到这里了，下期再见。
