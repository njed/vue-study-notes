---
description: VNode的相关代码存放在src/core/vdom包下面。
---

# Vue之VNode浅析

## 简单理解Virtual DOM

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

![\_render&#x51FD;&#x6570;&#x8C03;&#x7528;&#x6808;](../.gitbook/assets/image%20%281%29.png)



## Virtual DOM的diff算法

在Vue内部\_render方法只负责产出VNode实例，更新视觉的工作是由\_update方法负责的。从下面的简化版的流程图可以看出在\_\_patch\_\_方法的内部主要调用了createElm和patchVnode两个函数。只有当旧虚拟节点不是真的dom元素并且新旧虚拟节点相似的情况下才会通过patchVnode函数进行比较更新。那createElm和patchVnode有什么区别呢？

* createElm会递归生成dom元素，然后一次性挂载；
* patchVnode会递归对比VNode，只有不同的虚拟节点才会被处理更新dom；

![](../.gitbook/assets/image%20%286%29.png)

### sameVnode函数

sameVnode函数主要用于判断两个vnode实例是否相似，它会优先判断key值，在编写Vue组件时非v-for情况下，一般不会设置key值，而key值的默认值是undefined，而undefined === undefined总是为真。**那使用v-for指令时，设置key值带来的好处是什么？**

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

