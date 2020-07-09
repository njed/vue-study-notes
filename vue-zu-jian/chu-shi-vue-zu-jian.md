# 初识Vue组件

## 如何编写Vue组件?

如果你已经入门，最简单的方式就是使用.vue格式文件来编写组件了。

### .vue文件

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

.vue格式的文件会被vue-loader处理，在工程的build文件夹下找到webpack.base.conf.js配置文件，在其中的module.rule配置项下应该能看到如下的配置项。

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

这正是对.vue文件配置的插件，[vue-loader](https://vue-loader.vuejs.org/zh/)是Vue官方提供的webpack打包插件。.vue文件只是Vue官方提供的语法糖，template会被解析成render函数。还提供了了更强大的style作用域特性。详情可参考[官方文档](https://vue-loader.vuejs.org/zh/)。

### .js文件

> 一般使用js文件编写组件，是直接通过render函数来实现。

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

直接使用render函数实现组件在编程上更加灵活，但是问题也是显而易见的，就是元素过多的话编码会很繁琐，没有使用模板语法更直观。所以要视情况而定。一般更高层的组件可以使用render函数实现。更底层使用模板语法实现（会有大量UI呈现）。

## 你编写的Vue组件是什么?

仔细看上面两种不同方式编写的组件，导出的不都是一个纯对象吗？

{% code title="Test.js" %}
```javascript
export default {
  ...
}
```
{% endcode %}



