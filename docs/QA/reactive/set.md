### Q: 说说$set、$delete的原理？

### A:
虽然Vue实现了响应式编程，但Vue不能感知`对象或数组上新增属性的变化`，Vue提供`$set`来让对象或数组新增的数据具有响应性。


```js
export function set (target: Array<any> | Object, key: any, val: any): any {
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}
```

set原理:
1. 如果是数组并且key是有效的下标，将数组长度改成原数组长度和key中的较大值，然后调用`splice`新增元素，因为Vue拦截了数组方法，调用`splice`后会让新增的元素具有响应性
2. 如果key在对象或者数组上，则做简单赋值就好
3. 接着获取对象或数组的观察者对象，如果不存在观察者对象，则说明这个数组或对象不是响应式的，还是赋值就行
4. 以上情况都不是就表明在对象上新增属性，通过`defineReactive`给新增的属性添加响应功能并通知更新

```js
export function del (target: Array<any> | Object, key: any) {
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.splice(key, 1)
    return
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid deleting properties on a Vue instance or its root $data ' +
      '- just set it to null.'
    )
    return
  }
  if (!hasOwn(target, key)) {
    return
  }
  delete target[key]
  if (!ob) {
    return
  }
  ob.dep.notify()
}
```

delete原理和set相差不多，相对更加简单：
1. 数组数据直接`splice`即可
2. key不是对象上的下标，直接返回
3. 删除对象上的属性，删除后如果该对象不是响应式对象，直接返回，若是响应式对象就通知依赖更新