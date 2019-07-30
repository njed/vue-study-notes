---
description: Vue.extend主要用于类继承。
---

# Vue.extend

> 使用基础 Vue 构造器，创建一个“子类”。参数是一个包含组件选项的对象。
>
> `data` 选项是特例，需要注意 - 在 `Vue.extend()` 中它必须是函数。

Vue.extend函数就是使用基础Vue构造器创建子类，但是通过源码可以发现一些“蹊跷“？因为在前面构建组件的initState过程中发现，还没执行initProps和initComputed函数，但是属性已经存在组件实例上了，当时百思不得其解，直到看了Vue.extend源码才发现问题所在。

**因为props和computed属性会在后面代理到组件实例上，因此在Vue.extend内部优先把他们定义到了继承类的原型上，避免了在每次生成组件实例的时候再通过Object.defineProperty定义一遍。**

**而且继承类做了缓存，，避免二次生成相同继承类。**

{% code-tabs %}
{% code-tabs-item title="src/core/global-api/extend.js" %}
```javascript
  /**
   * Class inheritance
   */
  Vue.extend = function (extendOptions: Object): Function {
    extendOptions = extendOptions || {}
    const Super = this
    const SuperId = Super.cid
    const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
    if (cachedCtors[SuperId]) {
      return cachedCtors[SuperId]
    }

    const name = extendOptions.name || Super.options.name
    if (process.env.NODE_ENV !== 'production' && name) {
      validateComponentName(name)
    }

    const Sub = function VueComponent (options) {
      this._init(options)
    }
    Sub.prototype = Object.create(Super.prototype)
    Sub.prototype.constructor = Sub
    Sub.cid = cid++
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    )
    Sub['super'] = Super

    // For props and computed properties, we define the proxy getters on
    // the Vue instances at extension time, on the extended prototype. This
    // avoids Object.defineProperty calls for each instance created.
    if (Sub.options.props) {
      initProps(Sub)
    }
    if (Sub.options.computed) {
      initComputed(Sub)
    }

    // allow further extension/mixin/plugin usage
    Sub.extend = Super.extend
    Sub.mixin = Super.mixin
    Sub.use = Super.use

    // create asset registers, so extended classes
    // can have their private assets too.
    ASSET_TYPES.forEach(function (type) {
      Sub[type] = Super[type]
    })
    // enable recursive self-lookup
    if (name) {
      Sub.options.components[name] = Sub
    }

    // keep a reference to the super options at extension time.
    // later at instantiation we can check if Super's options have
    // been updated.
    Sub.superOptions = Super.options
    Sub.extendOptions = extendOptions
    Sub.sealedOptions = extend({}, Sub.options)

    // cache constructor
    cachedCtors[SuperId] = Sub
    return Sub
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

