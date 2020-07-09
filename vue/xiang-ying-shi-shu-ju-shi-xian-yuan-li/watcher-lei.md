---
description: 响应式数据核心Watcher类。
---

# Watcher类

> Watcher类的主要工作是解析表达式，收集依赖项，当表达式值变化时触发回调，该类也被$watch和指令使用。

## Watcher类构造函数

Watcher类的构造函数主要做了如下工作：

1. 检查是否为组件渲染侦听器（vm.\_watcher）；
2. 初始化内部相关状态；
3. 解析表达式，内部有getter维护；
4. 初始化value值，由lazy状态决定；

Watcher内部维护了deps和newDeps两个数组，这两个数组共同维护了该侦听器的依赖项。**为什么要维护两组依赖呢？**

Watcher内部还维护了一个自然增长的id值。**这个id有什么用呢？**

{% code title="src/core/observer/watcher.js" %}
```javascript

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
```
{% endcode %}

### 表达式解析

Watcher对象是通过内部的get函数来更新和获取value值的，而get函数内部又是通过Watcher对象内部的getter函数来生成value值的。表达式解析就是要生成一个正确的getter函数。如果构造函数传进来的expOrFn本事就是函数，那就直接赋值给getter，否则就通过parsePath函数生成一个getter函数。

{% code title="src/core/util/lang.js" %}
```javascript
/**
 * Parse simple path.
 */
const bailRE = /[^\w.$]/
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```
{% endcode %}

## Watcher类核心方法

* get：执行getter和重新收集依赖；
* update：依赖变化时调用的函数；
* run：观察者内部实际执行的函数；
* deardown：从所有依赖项中移除该订阅；

### Watcher之get函数

该函数逻辑很简单，设置依赖收集的Dep.target，执行getter（执行getter的过程会进行依赖收集），清除重置该观察者的依赖项（清除旧依赖，重置新为新依赖）。

#### 思考为什么要重新收集依赖？

{% code title="src/core/observer/watcher.js" %}
```javascript
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
```
{% endcode %}

### Watcher之update函数

当一个依赖项变化时该函数会被调用，也就是说依赖列表的任何一个依赖项变化都会触发该函数执行。该函数会根据不同的状态执行不同的逻辑代码。涉及lazy、sync以及侦听器执行队列。

####  哪些侦听器设置了lazy？

#### 哪些侦听器设置了sync？

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

### Watcher之run函数

{% code title="src/core/observer/watcher.js" %}
```javascript
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
```
{% endcode %}







