# 计算属性（computed）

> 计算属性可以解决在模板中放入太多的逻辑会让模板过重且难以维护的问题。而且**计算属性是基于它们的响应式依赖进行缓存的。**

Vue内部会为每个计算属性创建一个Watcher对象，并配置为lazy: true。这些Watcher对象被组件私有属性\_computedWatchers维护。计算属性不会因此直接进行计算，会被延迟到使用的时候才会真正计算，而且缓存结果（依赖项变化才会重新计算），因此计算属性比直接使用方法在性能上有明显的优势。

{% code-tabs %}
{% code-tabs-item title="src/core/intance/state.js" %}
```javascript
const computedWatcherOptions = { lazy: true }

function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Vue内部对props和computed对象都做了特殊处理，就是在使用Vue.extend继承生成组件类的时候，会把计算属性优先代理（getter/setter）到组件原型上，可以避免在组件实例化（initComputed）的时候为每个实例调用Object.defineProperty。

{% code-tabs %}
{% code-tabs-item title="src/core/global-api/extend.js" %}
```javascript
function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

当计算属性为函数，或使用getter/setter且cache为真的情况下（是的，还可以传入cache），这个时候会通过createComptedGetter函数生成一个新的替代自定义函数或getter函数的函数，该函数内部就使用了\_computedWatchers对象获取对应Watcher对象来获取计算值。此时应该明白计算属性是如何实现缓存的了吧，答案就是计算属性对应的Watcher对象。

{% code-tabs %}
{% code-tabs-item title="src/core/instance/state.js" %}
```javascript
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : userDef
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : userDef.get
      : noop
    sharedPropertyDefinition.set = userDef.set
      ? userDef.set
      : noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}

function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

