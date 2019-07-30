# Vue依赖收集详解

对于Vue中的所谓的依赖收集，首先要弄明白什么是依赖，收集依赖做什么？如前面章节所述，Vue内部使用Dep类来表示依赖项，每个需要观察的对象属性都会对应一个Dep实例。Vue中有哪些系统属性和用户自定义的属性会被观察呢？

### Vue中的依赖是什么？

数据，数据还是数据？重要的事情说三遍，那Vue中哪些数据会被当作依赖呢？Vue自身维护的$attr和$listeners,以及用户自定义的data和props都会被当做依赖项。

* $attr;
* $listeners;
* data;
* props;

上述这些对象的属性都会生成一个对应的Dep对象

```javascript
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

如果有人问Vue是怎么实现依赖收集的？使用Vue技术栈开发的同学，大多都能说出get和set存取描述符。但是仅仅只知道这些的话，那和不知道（非V）的区别也没什么多大的差别。仔细看defineReactive函数，他其实为每个需要处理的属性都形成了一个闭包，并且在get函数中动态收集依赖，所以在每次获取该属性值的时候都会判断是否需要依赖收集。并且会

