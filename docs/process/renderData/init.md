## 初始化组件
例子中`App`组件通过`extend`生成子构造器，子构造器在`createComponentInstanceForVnode`方法中被实例化来渲染组件内的数据，这就又回到了`_init`方法，定义在`src/core/instance/init.js`中

?> 渲染组件数据部分逻辑和渲染组件相同，不再重复描述
```js
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```
```js
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
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
		// 返回父配置项和子配置项的合并
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
	}
  // ...
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  }
}
```
**核心要点:**
- 此时传入的`options._isComponent`为`true`，执行`initInternalComponent`方法。

## initInternalComponent
```js
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
	const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```
**核心要点:**
- 主要将子构造器继承的`options`和父`vnode`上的`监听事件`等合并到`$options`上

## mount
!> 这里不贴代码，定义在`src/platforms/web/entry-runtime-with-compiler.js`中

在`_init`方法调用后，组件自身调用了`$mount`方法，此时在例子`App`组件内我们编写了`template`，`template`模板会被编译成`render`函数，详情点击[这里]()

## mountComponent

!> 这里不贴代码，定义在`src/core/instance/lifecycle.js`中

同样，这段代码的核心还是执行`updateComponent`函数，即`vm._update(vm._render(), hydrating)`这条语句。