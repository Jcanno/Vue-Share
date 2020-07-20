### Q: Vue实例的事件方法是怎么实现的？

### A:

常用的事件方法有`$on`、`$off`、`$once`、`$emit`


**$on:**

```js
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
  const vm: Component = this
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$on(event[i], fn)
    }
  } else {
    (vm._events[event] || (vm._events[event] = [])).push(fn)
  }
  return vm
}
```

`$on`方法支持传入事件名称和事件监听方法两个参数，如果事件名称是数组就遍历调用`$on`方法，否则就将监听方法储存到实例的`_events`数组上，`_events`在初始化时被赋值
```js
vm._events = Object.create(null)
```

**$off:**

```js
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {
  const vm: Component = this
  // all
  if (!arguments.length) {
    vm._events = Object.create(null)
    return vm
  }
  // array of events
  if (Array.isArray(event)) {
    for (let i = 0, l = event.length; i < l; i++) {
      vm.$off(event[i], fn)
    }
    return vm
  }
  // specific event
  const cbs = vm._events[event]
  if (!cbs) {
    return vm
  }
  if (!fn) {
    vm._events[event] = null
    return vm
  }
  // specific handler
  let cb
  let i = cbs.length
  while (i--) {
    cb = cbs[i]
    if (cb === fn || cb.fn === fn) {
      cbs.splice(i, 1)
      break
    }
  }
  return vm
}
```

`$off`中有许多情况，主要对传入的参数做适配:
1. 如果没有传入任何参数，就重置`_events`对象
2. 如果传入的事件名称是数组，就遍历递归调用`$off`方法
3. 然后再通过`event`事件名称拿到之前储存的监听方法集合
4. 如果监听方法集合不存在，说明之前没有对该事件名称进行监听，直接返回即可
5. 如果没有传入要取消监听的方法，就将该事件名称下的所有事件重置`vm._events[event] = null`
6. 最后一种情况是两个参数都传了，那就遍历事件监听方法的数组，如果方法和传入的取消监听方法一样，删除即可。这里是通过从后向前遍历，如果从前向后遍历，删除数组元素会造成部分元素的遗漏。


**$once:**

```js
Vue.prototype.$once = function (event: string, fn: Function): Component {
  const vm: Component = this
  function on () {
    vm.$off(event, on)
    fn.apply(vm, arguments)
  }
  on.fn = fn
  vm.$on(event, on)
  return vm
}
```
`$once`其实是在`$on`的基础上「改装」实现的，Vue在这里重写`$on`的监听方法函数，让其在执行监听方法的同时取消事件绑定，达到只调用一次监听方法的目的。



**$emit:**

```js
Vue.prototype.$emit = function (event: string): Component {
  const vm: Component = this
  let cbs = vm._events[event]
  if (cbs) {
    cbs = cbs.length > 1 ? toArray(cbs) : cbs
    const args = toArray(arguments, 1)
    const info = `event handler for "${event}"`
    for (let i = 0, l = cbs.length; i < l; i++) {
      invokeWithErrorHandling(cbs[i], vm, args, vm, info)
    }
  }
  return vm
}
```

可以看到，`$emit`方法会将`_events`中对应的事件方法取出，再将`arguments`转化成一个真正的数组参数，最后遍历监听方法，使用`invokeWithErrorHandling`将参数传入触发监听方法，`invokeWithErrorHandling`提供了执行监听方法同时处理错误的功能。
