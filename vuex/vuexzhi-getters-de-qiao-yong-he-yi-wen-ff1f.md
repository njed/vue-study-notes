# Vuex之getters的巧用和疑问？

Vuex中getter是什么？扮演着什么样的角色？

Vuex 允许我们在 store 中定义“getter”（可以认为是 store 的计算属性）。就像计算属性一样，getter 的返回值会根据它的依赖被缓存起来，且只有当它的依赖值发生了改变才会被重新计算。

getter非常灵活，我们可以可以根据不同的写法既可以把它当作属性使用（常规操作）。

```text
getter: (state) => state.items
```

我们也可以把它当作函数来调用？但是写法略有不同，我们需要返回一个函数。

```text
getter: (state) => {
    return (key) => {
        return state.items[key]
    }
}
```

在实际的开发过程中，我们前端从后端接口拿到的数据一般都需要经过处理才能使用。我们可以在vuex的state中保存原始数据，对需原始数据通过getter加工处理返回。

思考：getter

