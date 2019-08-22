# Vue源码分析

使用Vue库做了很多前端项目，也看过一些对源码分析的文章，但对于编写的Vue组件以及其内部运行原理的理解还很片面。如果问Vue的响应式数据是怎么实现的？嗯，使用了getter/setter实现。然后呢？没有然后了！通过工作之余对Vue源码的学习，来一步步揭开它神秘的面纱！

## 学习Vue源码，主要思考哪些内容？

* 组件实例化过程；
* 属性（data、props、hooks等等）合并策略；
* mixins和extends合并策略；
* 响应式数据实现原理；
* Virtual DOM、diff算法；

##  Vue源文件目录结构

对Vue源码分析内容大多包含在src/core目录下，这也是Vue的核心源码包，内容包括但不仅限于Vue官方实现的组件、Vue组件实例、Vue中的观察者以及vdom等。

* src/core/components
* src/core/global-api
* src/core/instance
* src/core/observer
* src/core/vdom

### src/core/instance/\*

> instance文件夹下的相关文件，涵盖了Vue的大部分实例函数实现，以及Vue组件的实例化过程等。

| 文件 | 说明 |
| :--- | :--- |
| events.js | initEvents, eventsMixin:\($on, $once, $off, $emit\) |
| index.js | Vue入口文件 |
| init.js | initMixin\(\_init\) |
| inject.js | initProvide, initInjections |
| lifecycle.js | initLifecycle, lifecycleMixin\(\_update, $forceUpdate, $destroy\), mountComponent, updateChildComponent, activateChildComponent, deactivateChildComponent, callHook |
| proxy.js |  |
| render.js | initRender, renderMixin\(renderList, renderSlot, renderStatic, ..., $nextTick, \_render\) |
| state.js | initState, defineComputed, stateMixin\($data,$props,$set,$delete,$watch\) |

### src/core/observer/\*

> observer文件夹包含了实现Vue响应式数据的核心文件，包括Watcher构造函数等。

| 文件 | 说明 |
| :--- | :--- |
| array.js | arrayMethods |
| dep.js | Dep构造函数, pushTarget, popTarget |
| index.js | Observer构造函数, observe, defineReactive |
| scheduler.js | queueWatcher |
| traverse.js | traverse |
| watcher.js | Watcher构造函数 |

