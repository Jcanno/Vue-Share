## 实例化Vue
根据我们的例子，例子中编写了`render`函数，`render`函数里传入了`App`组件，Vue在`渲染组件`和`渲染组件数据`过程是不一样的。我们先来看Vue在渲染组件上的工作。
```js
// src/core/instance/index
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
...
```
实例化Vue会执行`_init`方法，`_init`方法定义在`src/core/instance/init`中
```js
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
		// a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }
    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
		}
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
		}
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      vm._name = formatComponentName(vm, false)
      mark(endTag)
      measure(`vue ${vm._name} init`, startTag, endTag)
    }

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

**核心要点:**

- 有一段合并配置的代码，在首次调用时`options && options._isComponent`为`false`，因此调用`mergeOptions`方法合并配置
- 在实例上初始化了有关各模块一系列的属性(这里我们先大致了解，暂不做详细展开)
	1. initLifecycle: 初始化了有关生命周期的属性
	2. initEvents: 初始化了有关事件机制的属性
	3. initRender: 做了有关渲染`vnode`的属性和函数的初始化
	4. initInjections: 初始化注入属性`(inject和provide需要结合同时使用，一般用于高阶组件开发中，provide提供属性，inject负责注入获取属性，一般的程序开发我不需用到)`
	5. initState: 做了有关数据状态的初始化，劫持并监听数据的改变，实现数据的响应变化
	6. initProvide: 将`provide`属性添加到实例上，提供`inject`所需要的数据源
- 最终调用了`$mount`方法

## mergeOptions
`mergeOptions`定义在`src/core/util/options.js`中
```js
/**
 * Merge two option objects into a new one.
 * Core utility used in both instantiation and inheritance.
 */
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child, vm)
  normalizeInject(child, vm)
  normalizeDirectives(child)

  // Apply extends and mixins on the child options,
  // but only if it is a raw options object that isn't
  // the result of another mergeOptions call.
  // Only merged options has the _base property.
  if (!child._base) {
    if (child.extends) {
      parent = mergeOptions(parent, child.extends, vm)
    }
    if (child.mixins) {
      for (let i = 0, l = child.mixins.length; i < l; i++) {
        parent = mergeOptions(parent, child.mixins[i], vm)
      }
    }
  }

  const options = {}
  let key
  for (key in parent) {
    mergeField(key)
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }

  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```
首次调用`mergeOptions`时，`parent`相当于是`Vue.options`。`Vue.options`会在`initGlobalAPI`也就是我们之前说过的方法里定义，我们先不详细展开。这里是先将`mixins`合并到`parent`中，再遍历合并`parent`到最终的`options`，最后遍历`child`中的属性，将`child`特有的配置属性通过策略合并到`options`。这里都会调用`mergeField`方法，这个方法会根据不同的键名采用不同的合并策略，默认合并策略是优先使用`child`对应的值，否则使用`parent`的配置。

## 总结
这一节我们看到Vue实例化时在`_init`方法中会将Vue本身提供的配置和我们传入的配置进行合并，某些属性会有独自的合并策略。之后`_init`方法会初始化各模块的属性，最终会根据我们传入的`el`l来调用`$mount`方法来实现实例的挂载。