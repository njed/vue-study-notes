---
description: 主要介绍和分析如何跨组件传递slot内容。
---

# slot如何跨组件传递

## slot是什么?

> [`<slot>`](https://cn.vuejs.org/v2/api/?#slot) 元素作为组件模板之中的内容分发插槽。`<slot>` 元素自身将被替换。

## slot跨组件传递

跨组件的slot的引用场景，一般为包装组件\(CompWrapper\)包装了具体的组件\(Comp\)，一提供个性化的功能，而又不想丢失Comp提供的slot能力，那就需要CompWrapper提供slot穿透的能力，将slot透传给Comp。

### template语法实现

{% code-tabs %}
{% code-tabs-item title="CompA.vue" %}
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
  name: 'CompA'
}
</script>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### 如何让组件CompB支持组件CompA的slot？

{% code-tabs %}
{% code-tabs-item title="CompB.vue" %}
```javascript
<template>
  <comp-a>
    <!-- 传递default slot -->
    <slot></slot>
    <!-- 传递test slot -->
    <slot name="test" slot="test"></slot>
  </comp-a>
</template>

<script>
import CompA from './CompA'

export default {
  name: 'CompB',
  
  components: {
    CompA 
  }
}
</script>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

跨组件传递默认slot：很简单，和正常的一样，只是此时CompB会丧失自身默认slot的能力，因为父组件向CompB组件传递的默认slot其实是传递给了CompA；

跨组件传递具名slot：具名slot的此时的声明比较特殊，需要同时使用name和slot字段，name就是普通的slot名字，slot字段此时对应CompA的具名slot。在CompB中声明的跨组件slot名称一般和CompA中的slot名称保持一致；

{% code-tabs %}
{% code-tabs-item title="CompC.vue" %}
```javascript
<template>
  <comp-b>
    <span>default slot</span>
    <span slot="test">test slot</span>
  </comp-b>
</template>

<script>
import CompB from './CompB'

export default {
  name: 'CompC',
  
  components: {
    CompB 
  }
}
</script>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

组件CompB跨组件的slot，对于父组件CompC而言是无感知，在组件CompC中正常的向组件CompB传递slot，只是传递的slot会穿透CompB传递给组件CompA。

### render函数实现

如上使用template语法实现了slot跨组件传递，那如果使用render函数来实现，又该如何处理呢？

#### 使用render函数改造CompB函数

{% code-tabs %}
{% code-tabs-item title="CompB.vue" %}
```javascript
<script>
import CompA from './CompA'

export default {
  render (h) {
    h(CompA, [
      // default slot
      this._t('defualt'),
      // test slot
      this._t('test', null, { slot: 'test' })
    ])
  }
}
</script>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### 理解vm.\_t函数

> \_t函数是Vue内部渲染slot的私有的运行时函数。

{% code-tabs %}
{% code-tabs-item title="src/core/instance/render-helpers/render-slot.js" %}
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
{% endcode-tabs-item %}
{% endcode-tabs %}

