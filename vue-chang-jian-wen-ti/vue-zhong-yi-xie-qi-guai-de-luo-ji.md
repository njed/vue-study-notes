# Vue中v-if、v-else和v-for的关系

## v-if、v-else和v-for的优先级谁更高?

下面这段代码是不推荐的写法，而且它也不会正确渲染。执行结果：[jsfiddle](https://jsfiddle.net/njed_ann/xLfzsjte/4/)

```javascript
<template>
<div id="app">
    <span v-if="i < 3" v-for="i in 3" :key="i">{{i}}&nbsp;</span>
    <span v-else v-for="j in 3" :key="j">{{j}}&nbsp;</span>
</div>
</template>
```

**首先v-if和v-for指令一起使用时，v-for优先级高于v-if指令（官方文档也不推荐一起使用，推荐先过滤数组）。当v-else（要有前置的v-if）和v-for指令一起使用时，v-else的优先级是高于v-for的，为什么？**

但是上面的代码的渲染结果并不是1 2 1 2 3，而是1 2 undefined！！！一脸懵逼，undefined是怎么回事？

### 怎么看v-else的优先级高于v-for？

其实看一下模板对应生成的render函数就能得到答案。上面的模板生成的render函数如下：

```javascript
var render = function() {
  var _vm = this
  var _h = _vm.$createElement
  var _c = _vm._self._c || _h
  return _c(
    "div",
    { attrs: { id: "app" } },
    _vm._l(3, function(i) {
      return i < 3
        ? _c("span", { key: i }, [_vm._v(_vm._s(i) + " ")])
        : _vm._l(3, function(j) {
            return _c("span", { key: j }, [_vm._v(_vm._s(j) + " ")])
          })
    }),
    0
  )
}
```

从上面代码看到当i等于3的时候会通过\_vm.\_l生成包含三个元素的数组且tag是span，等等，不是说渲染结果是1 2 undefined吗，要打自己的脸？让我静一会，肯定哪里处理问题，render函数是干什么的，生成VNode实例的，但是VNode实例不是和真实dom对应的吗？那看下传入\_update函数的vnode到底怎么了？

![\_render&#x51FD;&#x6570;&#x751F;&#x6210;&#x7684;VNode&#x5B9E;&#x4F8B;](../.gitbook/assets/image%20%2811%29.png)

没毛病，是生成了五个VNode实例啊，等等，children的第三个元素是数组，发现情况有点不妙！莫不是这种数据格式有问题，再进一步跟踪下：

![createElm&#x51FD;&#x6570;&#x5185;&#x90E8;&#x903B;&#x8F91;](../.gitbook/assets/image%20%2810%29.png)

当对children进行遍历更新的时候，额，你竟然是个数组，我不认得你啊。vnode.text不就是undefined嘛！我没错！！！好吧，\_render函数没问题，能按照代码逻辑生成VNode实例和子节点实例，但是\_update函数不支持啊。

### 针对上面问题，怎么编写能正确渲染？

> 下面代码虽然能正确渲染出结果，也不推荐，不推荐，不推荐！引起不必要误会的代码逻辑都不是好逻辑！

```javascript
<template>
  <div id="app">
    <!-- <span v-for="item in items" :key="item.value">{{item.label}}</span> -->
    <span v-if="i < 3" v-for="i in 3" :key="i">{{i}}&nbsp;</span>
    <span v-else>
      <span v-for="j in 3" :key="j">{{j}}&nbsp;</span>
    </span>
  </div>
</template>
```

**把绑定v-else的span元素换成template也是不行的，因为并不会改变模板对应生成的render函数。**

### 总结

* v-if和v-for一起用，v-for优先级更高；
* v-else（有前置v-if）和v-for一起使用，v-else优先级更高；

