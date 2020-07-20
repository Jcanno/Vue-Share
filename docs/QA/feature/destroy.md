### Q: Vue实例销毁都做了什么工作？

### A:

```js
Vue.prototype.$destroy = function () {
  const vm: Component = this
  if (vm._isBeingDestroyed) {
    return
  }
  callHook(vm, 'beforeDestroy')
  vm._isBeingDestroyed = true
  // remove self from parent
  const parent = vm.$parent
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    remove(parent.$children, vm)
  }
  // teardown watchers
  if (vm._watcher) {
    vm._watcher.teardown()
  }
  let i = vm._watchers.length
  while (i--) {
    vm._watchers[i].teardown()
  }
  // remove reference from data ob
  // frozen object may not have observer.
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--
  }
  // call the last hook...
  vm._isDestroyed = true
  // invoke destroy hooks on current rendered tree
  vm.__patch__(vm._vnode, null)
  // fire destroyed hook
  callHook(vm, 'destroyed')
  // turn off all instance listeners.
  vm.$off()
  // remove __vue__ reference
  if (vm.$el) {
    vm.$el.__vue__ = null
  }
  // release circular reference (#6759)
  if (vm.$vnode) {
    vm.$vnode.parent = null
  }
}
```

主要工作有：
1. 通过`_isBeingDestroyed`标识来防止重复调用
2. 触发`beforeDestroy`生命周期函数
3. 在父实例中移除本实例
4. 将组件级的依赖`_watcher`移除
5. 将用户编写的依赖`_watchers`如`$watch`遍历移除
6. 调用`__patch__`触发更新，让Vue对比新老节点
7. 触发`destroyed`生命周期函数
8. 取消实例下所有事件监听方法
9. 移除`__vue__`引用
10. 将`$vnode.parent`赋为空，放置重复引用
