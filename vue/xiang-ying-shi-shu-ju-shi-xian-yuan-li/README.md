---
description: 学习和分析Vue内部的响应式数据的实现原理和相关知识点，以及相关设计模式和编程思想。
---

# 响应式数据实现原理

> 响应式数据是Vue 最独特的特性之一，是其非侵入性的响应式系统。数据模型仅仅是普通的 JavaScript 对象。而当你修改它们时，视图会进行更新。

非侵入式、普通的对象模型、动态更新试图，这些特性看起来非常的高大上，得益于Vue的强大响应式系统，减少了很多平常开发中的繁琐处理。把关注点从dom转移到了数据上，我们只需要维护数据状态，Vue就主动帮我们完成了剩下的工作。回想jquery盛行时期，当时的我们还需要手动去操作dom。那前端的下一次技术进化会是什么呢？

### 实现响应式数据的关键技术点

我们大致可以从以下几个方面进行思考和入手？

1. 如何对数据进行代理拦截？
2. 如何维护网状的关联关系？
3. 如何高性能的更新组件？

如果能够想明白上述的三个问题，其实大概也能实现一个响应式系统，下面就从Vue源码出发u 学习和分析下实现细节，学习下大牛们的编程思想。

### 通过存取描述符对数据进行代理拦截

还记得Object.defineProperty\(\)方法吗，Vue3之前的版本代理拦截操作正是基于属性存取描述符的get和set实现的（这里我们需要复习一下“[属性描述符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)”）,并在get函数中完成依赖收集\(depend\)并在set函数中触发更新通知\(notify\)。

思考：如何避免不必要的依赖收集？

{% code title="src/core/observer/index.js" %}
```javascript
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
{% endcode %}

总结一下defineReactive函数的作用：

* 每个被观察的属性都会形成一个闭包；
* 每个被观察的属性对应个一个Dep对象；
* 依赖收集是动态的，只有在真正使用到时才会进行依赖绑定；
* 重新赋值的时候会对属性值添加观察（如果有必要）；

Vue3应该会采用新规范的Proxy实现数据的代理拦截，Proxy在性能比采用属性存取描述符有大幅的提升。让我们一起期待Vue3的到来吧！

### 依赖收集：Dep类和Watcher类

首先要弄明白几个概念：观察者和被观察者。

被观察者：数据源，如：data和props；

观察者：依赖于这些数据源动态改变和计算的选项。如：watch和computed；

在Vue中观察者和被观察者是多对多的数据模型，每个数据源的属性都会对应一个Dep对象，每个观察者都会对应一个Watcher对象。

#### 依赖项：Dep类

首先要知道在Vue中哪些数据会被代理，Vue组件的$attrs和$listeners以及用户自定义属性data和props。Vue会对以上对象进行遍历，通过defineReactive函数对每个属性进行代理拦截。

在Vue内部使用Dep类表示依赖，每个被代理的属性都对应一个Dep对象，每个Dep对象内又保存了一个Watcher数组，在上述get函数中通过Dep对象的depend\(\)函数，双向绑定Dep和Watcher对象。从Dep类的设计上能明显的看出这是一个观察者模式的实现，当属性改变时通过Dep对象去通知Watcher对象进行更新操作。

{% code title="src/core/observer/dep.js" %}
```javascript
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
```
{% endcode %}

#### 依赖收集：Watcher类

Vue中的Watcher类用于解析表达式、收集依赖、当依赖项变化时触发回调。例如开发Vue组件时常用的watch和computed，他们都是通过对应的Watcher对象来维护依赖项和重新计算值的。具体对应关系如下：

* 每个watch选项会对应一个watcher对象；
* 每个computed选项会对应一个watcher对象；
* 组件内部会维护一个私有\_watcher对象用于更新组件；
* 组件内部会维护一个watcher数组，包括组件所有watcher对象；

当通过属性描述符set函数进行赋值时，内部会先进行一系列判断，如果值有变化最终会调用 Dep对象的notify\(\)方法通知依赖的观察者（Watcher对象）进行更新操作，而在Watcher对象内部会通过update\(\)函数进行后续一些列的更新操作。

{% code title="src/core/observer/watcher.js" %}
```javascript
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
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
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```
{% endcode %}

通过Watcher源码分析，能解释很多Vue官方的文档和api，例如$watch函数的内部实现，计算属性等等。留几个思考问题：

* $watch内部实现，以及返回的函数时怎么取消观察的？
* 计算属性是如何实现惰性计算和缓存的？

### Watcher更新队列

上一小节贴出的Watcher源码中如果仔细看update函数，会发现如果状态lazy为真的话，内部只是把dirty标记为true而已，并没有做任何别的事情，这个场景是和computed配合使用的；第二种情况是状态sync为真则直接调用run函数，run函数是真正执行计算和调用回调函数的地方；重点看下不满足上述两种的情况的时候其实是使用了队列来实现更新操作的。Vue的Watcher更新队列详情参考：Watcher更新队列章节。

{% code title="src/core/observer/watcher.js" %}
```javascript
  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

```
{% endcode %}

### 通过VNode优化DOM更新

