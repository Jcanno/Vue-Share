## 数据响应
Vue的数据是响应式的，核心是`数据的更改能引起视图的更新`。Vue中的数据响应由`Observe`类实现，通过`Object.defineProperty`来对数据的获取和赋值(即get和set)进行监控和劫持，在get和set劫持数据过程中添加方法达到数据响应的目的。我们通过源码来看数据响应的具体实现，Vue在初始化工作中就实现了数据响应，它发生在`initState`函数中，定义在`src/core/instance/init.js`。我们来看`initState`的实现，定义在`src/core/instance/state.js`。

```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
**核心要点:**
- 对各模块数据进行初始化，我们主要来看`initData`函数

```js
function initData (vm: Component) {
	let data = vm.$options.data
	
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
	}
  // observe data
  observe(data, true /* asRootData */)
}
```

**核心要点:**
- 获取`$options`中的data数据，即组件上的`data`对象
- 将`data`对象上的key与`methods`、`props`上的key做对比，判断是否存在重复命名
- 执行`observe`对数据进行观察

继续看`observe`函数的实现，定义在`src/core/observe/index.js`

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
**核心要点:**
- 获取缓存的`__ob__`观察者对象，若没有缓存，则实例化`Observer`

