### Q: keep-alive是怎么实现的？

### A:
`keep-alive`是Vue提供的内置的组件，在实践中一般我们会搭配`vue-router`中`router-view`一同使用，它能帮助我们缓存组件

```js
export default {
  name: 'keep-alive',
  abstract: true,

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },

  created () {
    this.cache = Object.create(null)
    this.keys = []
  },

  destroyed () {
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      const key: ?string = vnode.key == null
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode
        keys.push(key)
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}
```

`keep-alive`提供了三个属性，`include`、`exclude`、`max`，分别对应需要缓存的组件，不需要缓存的组件以及缓存的最大大小。


`keep-alive`是一个抽象组件，提供了render函数，主要逻辑也在render函数中。首先先获取了`keep-alive`中默认插槽，通过`getFirstComponentChild`函数获取到插槽中第一个组件，`getFirstComponentChild`主要是遍历插槽，再根据组件的特征取到第一个组件。

然后获取到组件名称，根据名称和传入的`include`、`exclude`来决定是否使用缓存，如果不需要缓存，则直接返回`vnode`。接着会通过组件的`key`拿到`keep-alive`持有的缓存对象，最后设置`vnode.data.keepAlive`为true

被`keep-alive`组件包裹的组件首次渲染和普通的首次渲染相同，`keep-alive`组件的render函数被执行，被包裹组件的`vnode.data.keepAlive`为true，当第二次渲染该组件时，patch阶段比对`vnode`时会先执行`prepatch`的钩子函数
```js
const componentVNodeHooks = {
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
  // ...
}
```
这里关键就是执行`updateChildComponent`方法，而这个方法会执行`$forceUpdate()`方法，就是强制让`keep-alive`组件重新render一遍，这个时候会根据`keep-alive`设置的标识，将之前的缓存`vnode`直接插入到DOM中，这就完成了缓存的工作。