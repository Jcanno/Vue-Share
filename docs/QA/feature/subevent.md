### Q: Vue怎么实现父子事件绑定？

### A:
Vue允许我们在HTML标签上绑定并监听事件，也允许我们在子组件上绑定自定义事件。在新老节点对比时会触发不同钩子函数，对于普通HTML元素，它会执行相关的事件绑定函数
```js
function updateDOMListeners (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (isUndef(oldVnode.data.on) && isUndef(vnode.data.on)) {
    return
  }
  const on = vnode.data.on || {}
  const oldOn = oldVnode.data.on || {}
  target = vnode.elm
  normalizeEvents(on)
  updateListeners(on, oldOn, add, remove, createOnceHandler, vnode.context)
  target = undefined
}
```
可以看到，当新老节点上都不存在事件时，函数直接返回，否则拿到新老节点事件绑定集合，通过`normalizeEvents`对事件做特殊处理，再调用`updateListeners`，更新事件
```js
export function updateListeners (
  on: Object,
  oldOn: Object,
  add: Function,
  remove: Function,
  createOnceHandler: Function,
  vm: Component
) {
  let name, def, cur, old, event
  for (name in on) {
    def = cur = on[name]
    old = oldOn[name]
    event = normalizeEvent(name)
    /* istanbul ignore if */
    if (__WEEX__ && isPlainObject(def)) {
      cur = def.handler
      event.params = def.params
    }
    if (isUndef(cur)) {
      process.env.NODE_ENV !== 'production' && warn(
        `Invalid handler for event "${event.name}": got ` + String(cur),
        vm
      )
    } else if (isUndef(old)) {
      if (isUndef(cur.fns)) {
        cur = on[name] = createFnInvoker(cur, vm)
      }
      if (isTrue(event.once)) {
        cur = on[name] = createOnceHandler(event.name, cur, event.capture)
      }
      add(event.name, cur, event.capture, event.passive, event.params)
    } else if (cur !== old) {
      old.fns = cur
      on[name] = old
    }
  }
  for (name in oldOn) {
    if (isUndef(on[name])) {
      event = normalizeEvent(name)
      remove(event.name, oldOn[name], event.capture)
    }
  }
}
```

`updateListeners`的逻辑就是遍历新事件集合添加事件，遍历老事件集合删除事件。

```js
function add (
  name: string,
  handler: Function,
  capture: boolean,
  passive: boolean
) {
  target.addEventListener(
    name,
    handler,
    supportsPassive
      ? { capture, passive }
      : capture
  )
}
```

`add`函数主要作用就是添加浏览器事件，这样就完成原生DOM事件的绑定。

对于组件上事件的绑定，假如有这样一个场景:
```js
{
	template: "<child @change='handleChange'></child>"
}
```
父组件监听子组件的change方法，在渲染子组件时，子组件初始化事件时会拿到父组件要求注册的事件。
```js
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```
这里执行了`updateComponentListeners`方法
```js
let target: any
export function updateComponentListeners (
  vm: Component,
  listeners: Object,
  oldListeners: ?Object
) {
  target = vm
  updateListeners(listeners, oldListeners || {}, add, remove, vm)
  target = undefined
}
```
同样是执行`updateListeners`的相关逻辑，但此时`add`不是添加浏览器事件，而是将事件添加到Vue的事件系统中

```js
function add (event, fn) {
  target.$on(event, fn)
}
```

子组件在`$emit`时就会触发父组件注册的事件