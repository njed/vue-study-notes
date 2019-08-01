---
description: 响应式数据核心Dep类。
---

# Dep类

Dep对象是可观察并可以有多个指令订阅它。每个可观察的对象属性会对应一个Dep对象，并且每个可观察的对象维护的Observer对象内部也会维护一个Dep对象。

Dep对象的作用就是在属性改变时通知观察者执行回调或更新视图。从Dep类的角度分析，它和Watcher类的关系就是观察者模式的实现。

**Dep类内部维护了一个全局自增长的id标示，他有什么作用呢？**

依赖收集也依赖于Dep.targat对象，同一时间只能有一个Watcher被评估，因此Watcher的get函数第一行代码就是pushTarget\(this\)，在属性的getter函数中再通过Dep.target判断来达到依赖收集的目的。

{% code-tabs %}
{% code-tabs-item title="src/core/observer/dep.js" %}
```javascript
let uid = 0

/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 */
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// the current target watcher being evaluated.
// this is globally unique because there could be only one
// watcher being evaluated at any time.
Dep.target = null
const targetStack = []

export function pushTarget (_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

export function popTarget () {
  Dep.target = targetStack.pop()
}



```
{% endcode-tabs-item %}
{% endcode-tabs %}

