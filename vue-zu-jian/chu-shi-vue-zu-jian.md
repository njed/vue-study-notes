---
description: 了解怎么编写Vue组件，以及Vue组件的本质。
---

# 初识Vue组件

## 如何编写Vue组件?

最简单的方式就是使用vue格式文件来编写组件了。Vue官方提供的[template语法糖](https://cn.vuejs.org/v2/guide/syntax.html)，可以让开发者像编写html一样编写Vue组件。当然如果你习惯使用原始的js，也可以使用render函数来实现组件。

### template语法

> Vue提供的编写组件的语法糖，最简单方便。template类似html标签语法。

{% code title="Test.vue" %}
```javascript
<template>
  <div class="test">Hello World!</div>
</template>

<script>
export default {
  name: 'Test'
}
</script>

<style scoped>
.test {
  color: red;
}
</style>
```
{% endcode %}

vue格式的文件会被vue-loader处理，如果使用vue-cli2.x版本生成的工程，在工程的build文件夹下找到webpack.base.conf.js配置文件，在其中的module.rule配置项下应该能看到如下的配置项。

{% code title="webpack.base.conf.js" %}
```javascript
{
  test: /\.vue$/,
  use: {
    loader: 'vue-loader',
    options: vueLoaderConfig
  }
}
```
{% endcode %}

这正是对vue文件配置的插件，[vue-loader](https://vue-loader.vuejs.org/zh/)是Vue官方提供的webpack打包插件。template只是Vue官方提供的语法糖，template会被解析成render函数。vue-loader还支持style的预处理器、作用域等特性。详情可参考[官方文档](https://vue-loader.vuejs.org/zh/)。

### render函数

> 直接使用render函数的话，我们可以使用vue文件或js文件来编写组件。

{% code title="Test.js" %}
```javascript
export default {
  name: 'Test',
  
  render (h) {
    return h('div', {
      'class': {
        test: true
      }
    }, ['Hello World!'])
  }
}
```
{% endcode %}

直接使用render函数实现组件在编程上更加灵活，但是问题也是显而易见的，就是元素过多的话编码会很繁琐，没有使用模板语法更直观。

**一般情况下即使是使用render函数实现组件，也推荐使用vue文件，这主要是涉及到样式问题，在js文件中是不能使用&lt;style&gt;标签的。**

## template和render如何选择？

使用template语法还是render函数来编写组件（排除个人爱好，部分大佬喜欢感受javascript的原始力量），应该视情况而定。使用template语法在展现上更加直观，修改也比较方便；而render在编码上则更加灵活，但是如果使用render函数实现复杂的视觉感觉是一场灾难。

**一般容器组件可以使用render函数实现，视觉组件使用模板语法实现。**

## 我们编写的是Vue组件吗？

如果仔细观察我们编写的组件，应该已经发现无论是使用template还是render函数，其实只是导出了一个配置对象而已。

{% code title="Test.vue" %}
```javascript
export default {
  ...
}
```
{% endcode %}

没错，我们编写的其实只是纯配置项而已，在运行时，组件创建函数createComponent会判断对应入参是配置对象还是构造函数，如果是配置项则会通过Vue.extend函数将配置对象转化成VueComponent构造器。 Vue.extend函数源码如下：

{% code title="src/core/global-api/extend.js" %}
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
{% endcode %}

### 直接导出Vue组件

{% code title="Test.vue" %}
```javascript
<script>
import Vue from 'vue'
export default Vue.extend({
  ...
})
</script>
```
{% endcode %}

直接导出Vue组件构造函数并没有明显的性能提升，主要是因为在Vue.extend中对组件构造函数做了缓存；

另外在编写“Vue组件“时使用Vue.extend直接导出组件构造函数也会容易产生不必要的误解（导出组件构造函数和组件选项对象有什么区别？），建议仅在编写组件的快捷调用函数时使用Vue.extend生成组件构造函数，再创建组件实例，然后挂载dom，例如$alert、$toast快捷函数等。

```javascript
import Alert from './Alert'

export default function (Vue) {
  const AlertConstructor = Vue.extend(Alert)
  // 全局保存一个Alert实例
  let instance = null
  Vue.prototype.$alert = function (message) {
    return new Promise((resolve, reject) => {
      if (!instance) {
        instance = new AlertConstructor({
          data: {
            message
          }
        })
        instance.$mount()
      } else {
        instance.setMessage(message)
      }
      document.body.appendChild(instance.$el)

      instance.$on('cancel', function () {
        // 可以remove el
        instance.$el.remove()
        reject(new Error())
      })
      instance.$on('confirm', function () {
        instance.$el.remove()
        resolve()
      })
    })
  }
}

```

