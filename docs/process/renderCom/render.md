## render

```js
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  if (_parentVnode) {
    vm.$scopedSlots = normalizeScopedSlots(
      _parentVnode.data.scopedSlots,
      vm.$slots,
      vm.$scopedSlots
    )
  }
  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    // There's no need to maintain a stack because all render fns arecalled
    // separately from one another. Nested component's render fns arecalled
    // when parent component is patched.
    currentRenderingInstance = vm
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    handleError(e, vm, `render`)
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production' && vm.$optionsrenderError) {
      try {
        vnode = vm.$options.renderError.call(vm._renderProxy, vm$createElement, e)
      } catch (e) {
        handleError(e, vm, `renderError`)
        vnode = vm._vnode
      }
    } else {
      vnode = vm._vnode
    }
  } finally {
    currentRenderingInstance = null
  }
  // if the returned array contains only a single node, allow it
  if (Array.isArray(vnode) && vnode.length === 1) {
    vnode = vnode[0]
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Renderfunction ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = createEmptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```
**核心要点:**
- `vnode = render.call(vm._renderProxy, vm.$createElement)`，调用`$createElement`方法生成`vnode`
- 将生成的`vnode`返回作为`_update`的参数

我们来看`$createElement`方法的实现，定义在`src/core/instance/render.js`中。
```js
// bind the createElement fn to this instance
// so that we get proper render context inside it.
// args order: tag, data, children, normalizationType, alwaysNormalize
// internal version is used by render functions compiled from templates
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
// normalization is always applied for the public version, used in
// user-written render functions.
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```
`$createElement`用于渲染用户手写的`render`函数，`_c`用于编译生成的`render`函数，它们都调用了`createElement`方法，区别是`规范化children`的方式不同。

`render`函数我们通常写法如下，React也是类似的写法
```js
h => h('div', {}, 'hello world')
// 这里的h即为$createElement，我们就确定了传入的参数
```
在我们的例子中，`render`函数传入了一个组件，组件生成`vnode`和非组件生成`vnode`过程又有点不同，我们来看`createElement`方法的实现，定义在`src/core/vdom/create-element.js`

```js
// wrapper function for providing a more flexible interface
// without getting yelled at by flow
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
	if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
	}
  return _createElement(context, tag, data, children, normalizationType)
}
```
**核心要点:**
- 设置规范化`children`方式
- 调用了`_createElement`方法
  
`createElement`实际上是在调用`_createElement`方法，在我们的例子中，首次这里传入的参数`tag`、`data`、`children`分别为`{ template: '<div>hello world</div>' }`、`undefined`、`undefined`，继续看`_createElement`的实现

## _createElement
```js
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
		ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)

    if (config.isReservedTag(tag)) {
      // platform built-in elements
      if (process.env.NODE_ENV !== 'production' && isDef(data) && isDef(data.nativeOn)) {
        warn(
          `The .native modifier for v-on is only valid on components but it was used on <${tag}>.`,
          context
        )
      }
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if ((!data || !data.pre) && isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
	}
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```
**核心要点:**
- 按照我们的例子，这里传入的`tag`是一个对象，`Vue中的组件本质就是一个对象`，因此在对`tag`的类型判断中会走`else`的逻辑，也就是通过调用`createComponent`来创建组件生成`vnode`。

## createComponent
`createComponent`定义在`src/core/vdom/create-component.js`中
```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```
**核心要点:**
- 定义了`baseCtor`，调用`baseCtor`的`extend`方法，详情点击[这里](process/renderCom/render?id=extend)
- 执行`installComponentHooks`函数，给组件安装钩子函数，详情点击[这里](process/renderCom/render?id=安装组件钩子)
- 通过`new VNode`生成`vnode`并返回

## extend

`baseCtor`使用的是`context.$options._base`，`_base`在初始化全局api时就被赋值，在`src/core/global-api`中有一条语句`Vue.options._base = Vue`，之后在`mergeOptions`合并到`$options`上，这里的`_base`就是`Vue`构造函数。

然后通过`baseCtor`调用了`extend`，这个方法也是在初始化全局api中定义好了，定义在`src/core/global-api/extend.js`中。

```js
Vue.extend = function (extendOptions: Object): Function {
  extendOptions = extendOptions || {}
  const Super = this
  const SuperId = Super.cid
  const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
  if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
  }
  const name = extendOptions.name || Super.options.name
  if (process.env.NODE_ENV !== 'production' && name) {
    validateComponentName(name)
  }
  const Sub = function VueComponent (options) {
    this._init(options)
  }
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  Sub.cid = cid++
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super
  // For props and computed properties, we define the proxy getters on
  // the Vue instances at extension time, on the extended prototype. This
  // avoids Object.defineProperty calls for each instance created.
  if (Sub.options.props) {
    initProps(Sub)
  }
  if (Sub.options.computed) {
    initComputed(Sub)
  }
  // allow further extension/mixin/plugin usage
  Sub.extend = Super.extend
  Sub.mixin = Super.mixin
  Sub.use = Super.use
  // create asset registers, so extended classes
  // can have their private assets too.
  ASSET_TYPES.forEach(function (type) {
    Sub[type] = Super[type]
  })
  // enable recursive self-lookup
  if (name) {
    Sub.options.components[name] = Sub
  }
  // keep a reference to the super options at extension time.
  // later at instantiation we can check if Super's options have
  // been updated.
  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  Sub.sealedOptions = extend({}, Sub.options)
  // cache constructor
  cachedCtors[SuperId] = Sub
  return Sub
}
```
这里就是通过经典原型继承方法生成了子构造器，`Sub`同样复制了`Super`上的全局方法。
```js
const Sub = function VueComponent (options) {
    this._init(options)
}
```
**核心要点:**
- 通过经典原型继承方法生成了子构造器，并且`Sub`复制了`Super`上的全局方法
- 我们先对`Sub`的定义有个印象，在后面组件初始化中会有用到

## 安装组件钩子

```js
function installComponentHooks (data: VNodeData) {
	const hooks = data.hook || (data.hook = {})
	for (let i = 0; i < hooksToMerge.length; i++) {
	  const key = hooksToMerge[i]
	  const existing = hooks[key]
	  const toMerge = componentVNodeHooks[key]
	  if (existing !== toMerge && !(existing && existing._merged)) {
	    hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
	  }
	}
}
const hooksToMerge = Object.keys(componentVNodeHooks)
const componentVNodeHooks = {
	init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
	  if (
	    vnode.componentInstance &&
	    !vnode.componentInstance._isDestroyed &&
	    vnode.data.keepAlive
	  ) {
	    // kept-alive components, treat as a patch
	    const mountedNode: any = vnode // work around flow
	    componentVNodeHooks.prepatch(mountedNode, mountedNode)
	  } else {
	    const child = vnode.componentInstance = createComponentInstanceForVnode(
	      vnode,
	      activeInstance
	    )
	    child.$mount(hydrating ? vnode.elm : undefined, hydrating)
	  }
	},
	prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
	  const options = vnode.componentOptions
	  const child = vnode.componentInstance = oldVnode.componentInstance
	  updateChildComponent(
	    child,
	    options.propsData, // updated props
	    options.listeners, // updated listeners
	    vnode, // new parent vnode
	    options.children // new children
	  )
	},
	insert (vnode: MountedComponentVNode) {
	  const { context, componentInstance } = vnode
	  if (!componentInstance._isMounted) {
	    componentInstance._isMounted = true
	    callHook(componentInstance, 'mounted')
	  }
	  if (vnode.data.keepAlive) {
	    if (context._isMounted) {
	      // vue-router#1212
	      // During updates, a kept-alive component's child components may
	      // change, so directly walking the tree here may call activated hooks
	      // on incorrect children. Instead we push them into a queue which will
	      // be processed after the whole patch process ended.
	      queueActivatedComponent(componentInstance)
	    } else {
	      activateChildComponent(componentInstance, true /* direct */)
	    }
	  }
	},
	destroy (vnode: MountedComponentVNode) {
	  const { componentInstance } = vnode
	  if (!componentInstance._isDestroyed) {
	    if (!vnode.data.keepAlive) {
	      componentInstance.$destroy()
	    } else {
	      deactivateChildComponent(componentInstance, true /* direct */)
	    }
	  }
	}
}
```
**核心要点:**
- 安装组件钩子，`init`方法在渲染组件内数据会被使用

## 总结
Vue中组件通过`createComponent`函数生成`vnode`，使用经典的原型继承的方式继承了父类构造器，并在组件实例上安装了钩子，用于初始化组件内数据，最后生成`vnode`返回。