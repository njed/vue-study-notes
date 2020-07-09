---
description: 响应式数据核心Observer类。
---

# Observer类

刚开始看Vue源码，当看到Watcher和Observer类的时候，**这是什么逻辑，怎么会有两个观察者类？**

1、其实Observer类并不是观察者模式中对应的观察者，Watcher类才是。在Vue中，Observer类会被附加到被观察的对象上，一旦连接，Observer类会将转化目标对象的属性为getter和setter用于依赖收集和更新。

2、被Observer类处理过的目标观察对象不会再次被观察处理。

3、当通过Vue.set为目标观察对象添加新属性时，会通过\_\_ob\_\_对象维护的Dep对象通知观察者更新视图。

{% code title="src/core/observer/index.js" %}
```javascript
/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through each property and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}

```
{% endcode %}

