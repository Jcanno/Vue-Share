## createElement

我们接着看`createElement`方法的实现，定义在`src/core/vdom/create-element.js`
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
这里我们还是寻找关键逻辑，因为我们在首次渲染时传入的`data`、`children`为`undefined`，前面的if逻辑先不用考虑，`tag`是一个对象，`Vue中的组件本质就是一个对象`，因此在`tag`的类型判断中会走`else`的逻辑，也就是通过调用`createComponent`来创建组件生成`vnode`。

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
这里的核心如下:
- 定义了`baseCtor`，调用`baseCtor`的`extend`方法

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
	这里的核心是`Sub`的定义,我们先对此有个印象，在后面组件初始化中会有用到
- 执行`installComponentHooks`函数，给组件安装钩子函数
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
	这是有关组件的钩子，我们需要关注的是`init`方法，这个方法会在之后使用到
- 通过`new VNode`生成`vnode`并返回
	最后通过实例化生成了`vnode`返回