# Vue源码分析

思考题

vue（类似框架）解决了实际开发中的什么难题？

响应式数据实现原理？怎么通知到所有引用了变化数据的组件？使用了什么策略？

vue中选项（data、props、methods、生命周期钩子）合并策略？

vue的混合和继承的合并策略？

data、props、methods、computed属性都会代理到vue实例上，重名会怎样？

vue中访问数组会发生什么？在循环中访问被vue观察的数组导致的性能问题？

什么是虚拟dom？使用虚拟dom对性能的影响？

diff算法？

nextTick执行的时机？

