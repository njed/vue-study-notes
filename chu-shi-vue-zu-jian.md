# 初识Vue组件

## 如何编写Vue组件?

如果你已经入门，最简单的方式就是使用.vue格式文件来编写组件了。

### .vue

{% code-tabs %}
{% code-tabs-item title="Test.vue" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

.vue格式的文件会被vue-loader处理，如果看过webpack的配置文件应该能找到下面的

```javascript

```

### .js

{% code-tabs %}
{% code-tabs-item title="Test.js" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

## 你编写的Vue组件是什么?

Yes, after a few months we finally found the answer. Sadly, Mike is on vacations right now so I'm afraid we are not able to provide the answer at this point.



