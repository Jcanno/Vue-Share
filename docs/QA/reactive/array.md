### Q: Vue如何处理数组方法监听数据变化？

### A:
Vue支持以下数组方法进行数据监听：
```js
[
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
```
这些方法的特点都能造成原数组元素的改变。
Vue还是通过拦截的方式在数组方法上做了手脚：
```js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```
处理方式为:
1. 获取array原型对象，并通过`Object.create`创建一个空对象
2. 遍历需要改造的数组方法即`methodsToPatch`
3. 在每个方法上改造成`mutator`函数，让数据调用目标方法时能触发`mutator`
4. `mutator`函数先调用缓存的原数组方法，就是让数组方法做原来该做的工作
5. 拿到每个数组的`__ob__`观察者对象
6. `push`、`unshift`、`splice`会新增元素，这些元素也需要响应检测，因此调用观察者对象上的`observeArray`，其原理就是遍历新增元素让他们成为响应式的
7. 最后就是通知依赖更新

`arrayMethods`被改造后能进行数组方法拦截了，现在就需要响应式的数组方法替换成我们改造后的`arrayMethods`对象

```js
import { arrayMethods } from './array'

const hasProto = '__proto__' in {}

export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

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
   * Walk through all properties and convert them into
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

function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}

function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

`Observer`类的作用是给所有数据添加响应功能，若传入的数据是数组类型，则将此数组方法替换成改造好的拦截器对象。

`Observer`中先判断当前浏览器环境是否支持`__proto__`，如果支持就可以直接将`arrayMethods`替换成数组原型对象(__proto__)，若不支持`__proto__`，就直接给数组数据添加改造的方法。