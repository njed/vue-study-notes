# Watcher更新队列

## Watcher更新

回顾一下Watcher类章节的内容，当依赖变化时会调用Watcher类的update方法，而此方法在判断状态非延迟和同步的情况下，会通过queueWatcher函数将自身添加到Watcher队列中。

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

## Watcher队列

Yes, after a few months we finally found the answer. Sadly, Mike is on vacations right now so I'm afraid we are not able to provide the answer at this point.

### queueWatcher函数

queueWatcher函数添加一个Watcher对象到队列中，如果队列空闲则直接添加队列后面；否则会遍历队列根据id值，把新增加的Watcher对象添加到合适的位置。然后判断是否需要刷新队列。队列是通过nextTick函数执行的（详情参考Vue.nextTick章节）。因此在队列真正被执行前可以继续添加Watcher对象。

{% code title="src/core/observer/scheduler.js" %}
```javascript
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```
{% endcode %}

### flushSchedulerQueue函数

在队列刷新函数中，首先会对Watcher队列做排序以确保：

* 先更新父组件，然后是自组件（因为父组件总是在自组件之前创建）；
* 组件内自定义Watcher优先于组件渲染Watcher执行（因为自定义Watcher在渲染Watcher之前被创建）；
* 父组件的Watcher执行期间销毁了某个子组件，可以跳过它的Watcher；

排序算法很简单，因为Watcher的id是从0自增长的，只需通过数组的sort简单排序即可。

排序完成就是遍历Watcher，执行其before和run方法。在开发模式下还会判断是否存在循环更新，判断逻辑是在不存在的情况下，该Watcher被遍历执行超过设置的最大阀值则任务存在循环（MAX\_UPDATE\_COUNT=100）。

{% code title="src/core/observer/scheduler.js" %}
```javascript
export const MAX_UPDATE_COUNT = 100

const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0

/**
 * Reset the scheduler's state.
 */
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}

/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```
{% endcode %}

## 组件activated事件钩子

在队列列表遍历结束后，会调用callActivatedHooks函数，执行"activated"事件钩子函数。

{% code title="src/core/observer/scheduler.js" %}
```javascript
/**
 * Queue a kept-alive component that was activated during patch.
 * The queue will be processed after the entire tree has been patched.
 */
export function queueActivatedComponent (vm: Component) {
  // setting _inactive to false here so that a render function can
  // rely on checking whether it's in an inactive tree (e.g. router-view)
  vm._inactive = false
  activatedChildren.push(vm)
}

function callActivatedHooks (queue) {
  for (let i = 0; i < queue.length; i++) {
    queue[i]._inactive = true
    activateChildComponent(queue[i], true /* true */)
  }
}
```
{% endcode %}

## 组件updated事件钩子

另外组件的“updated“事件钩子函数也是在flushSchedulerQueue函数中通过callUpdatedHooks函数触发的。

{% code title="src/core/observer/scheduler.js" %}
```javascript
function callUpdatedHooks (queue) {
  let i = queue.length
  while (i--) {
    const watcher = queue[i]
    const vm = watcher.vm
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'updated')
    }
  }
}
```
{% endcode %}

{% code title="src/core/instance/lifecycle.js" %}
```javascript
export function activateChildComponent (vm: Component, direct?: boolean) {
  if (direct) {
    vm._directInactive = false
    if (isInInactiveTree(vm)) {
      return
    }
  } else if (vm._directInactive) {
    return
  }
  if (vm._inactive || vm._inactive === null) {
    vm._inactive = false
    for (let i = 0; i < vm.$children.length; i++) {
      activateChildComponent(vm.$children[i])
    }
    callHook(vm, 'activated')
  }
}
```
{% endcode %}

