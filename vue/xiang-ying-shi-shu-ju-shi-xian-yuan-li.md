# 响应式数据实现原理

Vue的核心响应式数据，并动态改变ui呈现。看起来非常高大上，我们只需要改变属性值，Vue就主动帮我们完成了剩下的工作，回想jquery盛行时期，当时的我们还需要手动去操作dom。那前端的下一次技术进化会是什么呢？

### 如果我们要实现一个类Vue的响应式数据，我们应该怎么实现呢？

我们大致可以从以下几个方面进行思考？

1. 对数据进行拦截代理；
2. 收集数据依赖项；
3. 数据变化通知依赖项；
4. 依赖项动态改变dom完成更新操作；

知道了实现响应式数据的大致技术点之后，是不是觉得响应式数据也不是很神秘嘛！程序员的一个通病就是没动手之前都觉得问题很简单，开始编码之后要考虑各种细节了，突然觉得好难哦。言归正传，对于上述技术点我们就要考虑怎么对第三方数据进行拦截代理？又怎么收集依赖项等等，下面我们就Vue源码进行分析，看下大牛们是怎么思考和实现的。

### 通过存取描述符对数据进行拦截代理

还记得Object.defineProperty\(\)方法吗，Vue3之前的版本正是基于属性存取描述符的get和set实现的（这里我们需要复习一下“[属性描述符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)”）,并在get函数中完成依赖收集\(depend\)和在set函数中触发更新通知\(notify\)。

思考：如何避免不必要的依赖收集？

{% code-tabs %}
{% code-tabs-item title="src/core/observer/index.js" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

Vue3应该会采用新规范的Proxy实现数据的拦截代理，据说Proxy在性能比采用属性存取描述符有大幅的提升。让我们一起期待Vue3的到来吧！

### 依赖收集：Dep类

首先要知道在Vue中哪些数据会被代理，在编写Vue组件时，组件内部维护的数组包括自身data、外部传入的props以及vuex维护的数据，我们这里排除vuex那就剩下data和props了。Vue内部会遍历data和props对象，对属性defineReactive函数对每个属性进行代理。

在Vue内部使用Dep类表示依赖，在Vue中每个被代理的属性都对应一个Dep对象，每个Dep对象内又保存了一个Watcher数组，在上述get函数中通过Dep对象的depend\(\)函数，双向绑定Dep和Watcher对象。从Dep类的设计上能明显的看出这是一个观察者模式的实现，当属性改变时通过Dep对象去通知Watcher对象进行更新操作。

* data的每个属性对应一个Dep对象
* props的每个属性对应一个Dep对象；

{% code-tabs %}
{% code-tabs-item title="src/core/observer/dep.js" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

### 依赖收集：Watcher类

Vue中的Watcher类用于解析表达式、收集依赖、当依赖项变化时触发回调。最应该想到的就是常用的watch和computed，他们都是通过对应的Watcher对象来维护依赖项和重新计算值的。具体对应关系如下：

* 每个watch会对应一个watcher对象；
* 每个computed会对应一个watcher对象；
* 组件内部会维护一个私有\_watcher对象用于更新组件；
* 组件内部会维护一个watcher数组，包括组件所有watcher对象；

当为代理的属性赋值时，对应set函数内部会先进行一系列判断，如果值有变化最终会调用 Dep对象的notify\(\)方法通知依赖的观察者（Watcher对象），而在Watcher对象内部会通过update\(\)函数进行后续一些列的更新操作。

{% code-tabs %}
{% code-tabs-item title="src/core/observer/watcher.js" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

通过Watcher源码分析，能解释很多Vue官方的文档和api，例如$watch函数的内部实现，计算属性等等。留两个思考问题：

* $watch内部实现，以及怎么取消观察的？
* 计算属性是如何实现惰性计算和缓存的？

### Watcher更新队列

上一小节贴出的Watcher源码中如果仔细看update函数，会发现如果状态lazy为真的话，内部只是把dirty标记为true而已，并没有做任何别的事情，这个场景是和computed配合使用的；第二种情况是状态sync为真则直接调用run函数，run函数是真正执行计算和调用回调函数的地方；重点看下不满足上述两种的情况的时候其实是使用了队列来实现更新操作的。

{% code-tabs %}
{% code-tabs-item title="src/core/observer/watcher.js" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

### 通过VNode优化DOM更新

