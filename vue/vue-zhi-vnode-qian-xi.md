---
description: VNode的相关代码存放在src/core/vdom包下面。
---

# Vue之VNode浅析

## Vue中Virtual DOM的具体实现

Vue中和Virtual DOM相关的文件存放在src/core/vdom包下，有兴趣的话可自行查看。

Vue内部会维护一个虚拟DOM来对应Vue组件树，虚拟DOM的存在主要是为了优化更新真实DOM树的性能问题。在Vue组件中，当数据变化时组件如何最小化的更新真实DOM树是Vue要解决的主要性能问题。

而使用虚拟DOM的话，Vue可以在感知数据变化后生成新的虚拟DOM，并通过新旧虚拟DOM的比较（比较算法），把**差异**应用到真实DOM树上。

### VNode类

Vue内部使用VNode实例来维护虚拟节点，当数据变化时会通过vm.\_watcher来更新组件，更新组件时会先调用\_render方法来生成新的VNode实例，然后通过新旧虚拟节点的比较找出差异，并更新视觉。

{% code-tabs %}
{% code-tabs-item title="src/core/vdom/vnode.js" %}
```javascript
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

还记得在使用v-for指令时，

### VNode实例生成过程

![\_render&#x51FD;&#x6570;&#x8C03;&#x7528;&#x6808;](../.gitbook/assets/image%20%2810%29.png)



## Virtual DOM的diff算法

在Vue内部\_render方法只负责产出VNode实例，更新视觉的工作是由\_update方法负责的。从下面的简化版的流程图可以看出在\_\_patch\_\_方法的内部主要调用了createElm和patchVnode两个函数。只有当旧虚拟节点不是真的dom元素并且新旧虚拟节点相似的情况下才会通过patchVnode函数进行比较更新。那createElm和patchVnode有什么区别呢？

* createElm会递归生成dom元素，然后一次性挂载；
* patchVnode会递归对比VNode，只有不同的虚拟节点才会被处理更新dom；

![\_update&#x8C03;&#x7528;&#x6D41;&#x7A0B;&#x56FE;](../.gitbook/assets/image%20%284%29.png)

### Vue内部VNode更新操作

patchVnode函数主要通过对新旧虚拟节点的对比，找出需要更新、删除、新增和移动的节点执行对应的操作。下面以一个简单的列表删除操作来看下：

```javascript
<template>
  <div id="app">
    <span v-for="item in items" :key="item.value">{{item.label}}</span>
  </div>
</template>

<script>

export default {
  name: 'App',

  data () {
    return {
      items: [
        { label: '1', value: 1 },
        { label: '2', value: 2 },
        { label: '3', value: 3 }
      ]
    }
  }
}
</script>
```

如果针对上面列表执行选项删除操作，大致的行为如下图所示：

![&#x9009;&#x9879;&#x64CD;&#x4F5C;&#x6267;&#x884C;&#x8FC7;&#x7A0B;](../.gitbook/assets/image%20%283%29.png)

Vue内部通过Virtual Node的diff算法会对内存在的VNode实例进行对比，过滤掉相同节点，以减少不必要的dom更新操作。Vue内部实现主要涉及patchVnode、updateChildren和sameVnode三个主要的函数。

* patchVnode: 对比指定的新旧VNode节点实例，更新dom（需要的话）；
* updateChildren: 遍历新旧子VNode节点列表，并对新旧节点实例应用patchVnode；
* sameVnode: 判断两个VNode实例是否相同；

### sameVnode函数

sameVnode函数主要用于判断两个vnode实例是否相同，它会优先判断key值，在编写Vue组件时非v-for情况下，一般不会设置key值（此时key值为undefined，而undefined === undefined为真）。对比内容还包括tag、isComment、sameInputType等等。

**思考：使用v-for指令时，设置key值带来的好处是什么？**

{% code-tabs %}
{% code-tabs-item title="src/core/vdom/patch.js" %}
```javascript
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        !childrenIgnored(a) && !childrenIgnored(b) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### patchVnode函数

{% code-tabs %}
{% code-tabs-item title="src/core/vdom/patch.js" %}
```javascript
  function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
    if (oldVnode === vnode) {
      return
    }

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
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### updateChildren函数

```javascript
  function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
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
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(newCh)
    }

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue)
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
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

