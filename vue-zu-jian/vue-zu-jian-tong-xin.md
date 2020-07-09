---
description: 探讨分析组件通信的场景和不同解决方案。
---

# Vue组件通信

组件间通信是组织和关联不同组件的核心所在，一般包括数据传递、事件处理等。场景则包括父子组件间通信、兄弟组件间通信，跨多级组件间通信，下面就用简单的实例讲解各种场景的通信方案。

## 父子组件通信

父子组件间通信是最基础和常见的一种组件通信场景。实现通信的方式也很简单，父组件通过props向子组件传递数据，子组件通过$emit方法触发事件，父组件只需通过v-on监听事件即形成了一个闭环。下面就以简单的列表和选项组件为例说明：

{% code title="List.vue" %}
```javascript
<template>
<div class="list">
    <item v-for="item in items" :key="item.id" :item="item" @delete="handleDelete"/>
</div>
</template>
<script>
import Item from './Item'

export default {
    name: 'Item',

    components: {
        Item
    },

    data () {
        return {
            items: [
                { id: 'id_1', label: '测试数据1' },
                { id: 'id_2', label: '测试数据2' }
            ]
        }
    },
    methods: {
        handleDelete (item) {
            this.items = this.items.filter(({ id }) => id !== item.id)
        }
    }
}
</script>
```
{% endcode %}

{% code title="Item.vue" %}
```javascript
<template>
    <div class="item">
        <p>{{item.label}}</p>
        <button @click="$emit('delete', item)">删除</button>
    </div>
</template>
<script>
export default {
    name: 'Item',

    props: {
        item: {
            type: Object,
            required: true
        }
    }
}
</script>
```
{% endcode %}

**在Vue的状态管理中，不推荐在子组件内修改父组件的相关状态，因此对相关父组件状态的修改行为推荐通过触发相关事件交由父组件处理。这和react中直接向子组件传递action类似。**

## 兄弟组件通信

兄弟组件间通信很容易想到的一个场景就是省市区这种的关联性组件（省份组件值改变触发市组件列表的改变）。一般通过全局总线（维护的一个全局的Vue实例$bus）来实现通信，即通过$bus.$emit和$bus.$on来触发和监听事件。

{% code title="CompA.vue" %}
```javascript
<script>
export default {
    name: 'CompA',

    methods: {
        change () {
            this.$bus.$emit('a-value-change', 'test')
        }
    }
}
</script>
```
{% endcode %}

{% code title="CompB.vue" %}
```javascript
<script>
export default {
    name: 'CompB',

    created () {
        this.$bus.$on('a-value-change', this.handleChange)
    },

    methods: {
        handleChange (value) {
            console.log(value)
        }
    }
}
</script>
```
{% endcode %}

使用全局总线实现兄弟组件间通信的方式其实就是观察者模式的一种实现，不仅CompB可以监听CompA的事件“a-value-change“，其它任何组件都可以监听该事件并作出相应改变。

## 跨级组件通信

跨多级组件通信和兄弟组件间通信类似，一般可通过全局总线（$bus）实现；但对于高阶插件/组件库而言，可通过provide\inject实现数据的传递，但是事件逐级向上/向下传递的话，超过两级结构就会显得繁琐，此时我们可以实现dispatch和broadcase来优化事件的传递。

dispatch方法向上迭代直到找到指定的祖先组件，然后触发祖先组件的指定事件。

broadcase方法向下递归遍历对指定的子孙组件触发指定的事件。

这在实现类似RadioGroup和Radio组件组的时候很有用，在子孙组件Radio中可直接通过触发祖先组件RadioGroup的对应事件来完成时间通知。

{% code title="Radio.js" %}
```javascript
export default {
    name: 'Radio',
    componentName: 'Radio',
    ...,
    methods: {
        change () {
            this.$emit('change', this.value)
            this.isGroup && this.dispatch('RadioGroup', 'handleChange', this.value)
        }
    },
    render (h) {
        return h(...)
    }
}
```
{% endcode %}

{% code title="RadioGroup.js" %}
```javascript
export default {
    name: 'RadioGroup',
    componentName: 'RadioGroup',
    created () {
        this.$on('handleChange', this.handleChange)
    }
    methods: {
        handleChange (val) {
            ...
        }
    }
}
```
{% endcode %}

### 参考element-ui的实现

{% code title="emitter.js" %}
```javascript
function broadcast(componentName, eventName, params) {
  this.$children.forEach(child => {
    var name = child.$options.componentName;

    if (name === componentName) {
      child.$emit.apply(child, [eventName].concat(params));
    } else {
      broadcast.apply(child, [componentName, eventName].concat([params]));
    }
  });
}
export default {
  methods: {
    dispatch(componentName, eventName, params) {
      var parent = this.$parent || this.$root;
      var name = parent.$options.componentName;

      while (parent && (!name || name !== componentName)) {
        parent = parent.$parent;

        if (parent) {
          name = parent.$options.componentName;
        }
      }
      if (parent) {
        parent.$emit.apply(parent, [eventName].concat(params));
      }
    },
    broadcast(componentName, eventName, params) {
      broadcast.call(this, componentName, eventName, params);
    }
  }
};
```
{% endcode %}

## 总结

针对以上三种组件通信场景的解决方案，并不一定是最优的，不同的组件实现方式会有不同的通信解决方案。但是在实际产品的编码过程中以上几种方案确实能很好的解决所面临的问题。就像Vue本身高版本提供的provide/inject一样，面对问题要不停探索和优化。

