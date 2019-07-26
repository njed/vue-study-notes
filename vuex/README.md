# Vuex

先贴上Vuex的官方文档：[https://vuex.vuejs.org/zh/guide/structure.html](https://vuex.vuejs.org/zh/guide/structure.html)

对于Vuex的学习，我们可以先从一些几个问题出发：

## 1、vuex是什么？

Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。

## 2、解决了什么问题？

多组件共享状态。一般情况下我们通过传参的方式\(props\)进行父子间组件的状态传递，但是当组件嵌套很深时，代码调试和数据追踪就变成了我们的噩梦。

## 3、什么场景下适合使用vuex？

实际开发中并不是所有项目都适用vuex，毕竟要额外引入该插件，还要实际开发中代码写起来也是相当的繁琐冗余。

业务逻辑简单或者说不需要组件共享状态的应用并没有引用vuex的必要，我们使用vue组件中的data就能解决的问题干嘛为了使用vuex而使用vuex呢？

那什么场景下适用vuex呢？当我们需要构建一个中大型单页应用，您很可能会考虑如何更好地在组件外部管理状态，Vuex 将会成为自然而然的选择。

## 4、store是怎么注入到vue的每个组件中的？

Vuex在install的时候，通过Vue混入对象的方式在beforeCreate事件钩子中将$store属性注入到每个组件中。

```text
  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
```

从源码中我们可以看到，当我们在根组件中注入store属性后，后面创建的子组件都会将父组件的$store属性注入进来。

## 5、vuex状态树是怎么变成响应式数据的？

## 6、高阶函数在vuex中的应用？

