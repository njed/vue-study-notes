# 侦听器（watch）

> Vue中的[watch](https://cn.vuejs.org/v2/guide/computed.html#%E4%BE%A6%E5%90%AC%E5%99%A8)用来响应数据的变化。当需要在数据变化时执行异步或开销较大的操作时，这个方式是最有用的。

从下面的initWatch函数中可以监听器的回调函数可以是一个数组，并且在createWatcher内部也是通过$watch函数来添加监听器的。

{% code title="src/core/instance/state.js" %}
```javascript
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}

function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```
{% endcode %}

重点看下$watch函数内是如何实现添加监听器的。主要是通过Watcher类创建了一个观察者对象，并返回取消观察的unwatchFn函数，在看下unwatchFn函数，内部只是执行了前面生成的watcher.teardown函数。仔细看下代码，就能知道为什么存在immediate时，不能在第一次回调中取消侦听了，因为第一次回调是在$watch没返回之前执行的，此时unwatchFn并不存在。

{% code title="src/conre/instance/state.js" %}
```javascript
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    const vm: Component = this
    if (isPlainObject(cb)) {
      return createWatcher(vm, expOrFn, cb, options)
    }
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    return function unwatchFn () {
      watcher.teardown()
    }
  }

```
{% endcode %}

