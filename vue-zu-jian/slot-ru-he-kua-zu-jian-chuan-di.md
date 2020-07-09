---
description: 主要介绍和分析如何跨组件传递slot内容。
---

# slot如何跨组件传递

## slot是什么?

> [`<slot>`](https://cn.vuejs.org/v2/api/?#slot) 元素作为组件模板之中的内容分发插槽。`<slot>` 元素自身将被替换。

Vue的插槽详细说明和用法请参考[官方文档](https://cn.vuejs.org/v2/guide/components-slots.html)，本篇文章主要讨论跨组件传递slot的实现方式和用途。

## slot跨组件传递使用场景

跨传递slot一般的使用场景为包装组件需要保留原组件的slot功能。那么就需要CompWrapper跨组件传递slot了。

## slot跨组件传递实现方式

实现方式有多种，包括但不仅限于template、scopedSlots以及直接使用slot渲染函数实现。在版本2.6.0及以上，Vue对slot和scopedSlot做了统一，新增了v-slot指令。

### template语法实现

{% code title="Comp.vue" %}
```javascript
<template>
  <div>
    <!-- 默认slot -->
    <slot></slot>
    <!-- 具名slot -->
    <slot name="test"></slot>
  </div>
</template>

<script>
export default {
  name: 'Comp'
}
</script>
```
{% endcode %}

#### 如何让组件CompWrapper支持组件Comp的slot？

{% code title="CompWrapper.vue" %}
```javascript
<template>
  <comp>
    <!-- 传递default slot -->
    <slot></slot>
    <!-- 传递test slot -->
    <slot name="test" slot="test"></slot>
  </comp>
</template>

<script>
import Comp from './Comp'

export default {
  name: 'CompWrapper',
  
  components: {
    Comp
  }
}
</script>
```
{% endcode %}

跨组件传递默认slot很简单，只需将默认slot作为Comp的子节点即可，只是此时CompWrapper会丧失自身默认slot的能力，因为向CompWrapper组件传递的默认slot其实是传递给了Comp；

跨组件传递具名slot的声明比较特殊，需要同时使用name和slot字段，name就是普通的slot名字，slot字段此时对应Comp的具名slot。在CompWrapper中声明的跨组件slot名称一般和Comp中的slot名称保持一致；

{% code title="Test.vue" %}
```javascript
<template>
  <comp-wrapper>
    <span>default slot</span>
    <span slot="test">test slot</span>
  </comp-wrapper>
</template>

<script>
import CompWrapper from './CompWrapper'

export default {
  name: 'Test',
  
  components: {
    CompWrapper
  }
}
</script>
```
{% endcode %}

组件CompWrapper跨组件的slot，对于使用组件Test而言是无感知的，在组件Test中正常的向组件CompWrapper传递slot，只是传递的slot会穿透CompWrapper传递给组件Comp。

### scopedSlots实现

借助官方提供的scopedSlots，也可以很简单的实现跨组件传递slot，只是在生成对应slot的时候需要判断对应的slot函数是否存在，因为在使用组件中没有向CompWrapper传递slot的话，$scopedSlots中是不会存在对应slot函数。

{% code title="CompWrapper.vue" %}
```javascript
<script>
import Comp from './Comp'

export default {
  name: 'CompWrapper',
  
  render (h) {
    const ss = this.$scopedSlots
    
    h(CompA, {
      scopedSlots: {
        default (props) {
          return ss.default ? ss.default(props) : '插槽默认值'
        },
        test (props) {
          return ss.test ? ss.test(props) : '具名插槽默认值'
        }
      }
    })
  }
}
</script>
```
{% endcode %}

### slot渲染函数实现

slot渲染函数不推荐使用，因为是Vue内部的私有函数不能保证在后续的版本更新中是否保持一致，this.\_t也有可能改成其它函数名。如果我们使用template实现slot，Vue也是帮我们转化为下面的slot渲染函数。

{% code title="CompWrapper.vue" %}
```javascript
<script>
import Comp from './Comp'

export default {
  name: 'CompWrapper',
  
  render (h) {
    h(CompA, [
      // default slot
      this._t('defualt', ['插槽默认值'], { ... }),
      // test slot
      this._t('test', ['具名插槽默认值'], { ... })
    ])
  }
}
</script>
```
{% endcode %}

#### 理解vm.\_t函数

> \_t函数是Vue内部渲染slot的私有的运行时函数。

{% code title="src/core/instance/render-helpers/render-slot.js" %}
```javascript
/* @flow */

import { extend, warn, isObject } from 'core/util/index'

/**
 * Runtime helper for rendering <slot>
 */
export function renderSlot (
  name: string,
  fallback: ?Array<VNode>,
  props: ?Object,
  bindObject: ?Object
): ?Array<VNode> {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes
  if (scopedSlotFn) { // scoped slot
    props = props || {}
    if (bindObject) {
      if (process.env.NODE_ENV !== 'production' && !isObject(bindObject)) {
        warn(
          'slot v-bind without argument expects an Object',
          this
        )
      }
      props = extend(extend({}, bindObject), props)
    }
    nodes = scopedSlotFn(props) || fallback
  } else {
    const slotNodes = this.$slots[name]
    // warn duplicate slot usage
    if (slotNodes) {
      if (process.env.NODE_ENV !== 'production' && slotNodes._rendered) {
        warn(
          `Duplicate presence of slot "${name}" found in the same render tree ` +
          `- this will likely cause render errors.`,
          this
        )
      }
      slotNodes._rendered = true
    }
    nodes = slotNodes || fallback
  }

  const target = props && props.slot
  if (target) {
    return this.$createElement('template', { slot: target }, nodes)
  } else {
    return nodes
  }
}
```
{% endcode %}

## 解决CompWrapper组件自身slot的问题

因为上面的实例会让CompWrapper组件丧失提供自身默认slot的能力，解决方案很简单就是提供一个具名slot作为Comp的默认slot。

{% code title="CompWrappr.vue" %}
```javascript
<template>
  <comp>
    <!-- 传递default slot -->
    <slot name="subDefault" slot="default"></slot>
    <!-- 传递test slot -->
    <slot name="test" slot="test"></slot>
  </comp>
</template>

<script>
import Comp from './Comp'

export default {
  name: 'CompWrapper',
  
  components: {
    Comp
  }
}
</script>

```
{% endcode %}

