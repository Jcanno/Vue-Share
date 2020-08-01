### Q: 状态(数据)改变后都发生了什么?

### A:
数据发生变化，会触发相应的依赖更新，也就是状态的`dep`会执行持有每一个`watcher`里的`update`方法。

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```
`update`会执行`queueWatcher`方法将`watcher`实例推入队列中，`queueWatcher`中添加了防止重复添加`watcher`的逻辑。在当标记队列状态显示可以执行更新时，则会调用`nexttick`在下一个`tick`更新DOM，至于为什么使用下一个`tick`来更新DOM，这是Vue在更新DOM做的优化，这能让状态依赖(`watcher`)集中处理更新。`nexttick`中会将回调方法推入到一个数组中，然后使用`Promise`、`setTimeout`、`setImmediate`等方式兼容用户当前的环境来实现下一个`tick`。当到了下一个`tick`会执行数组里所有的回调，这些回调指的就是`watcher`的更新DOM的方法。

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
最终`nexttick`执行的回调函数实际上是`flushSchedulerQueue`，`flushSchedulerQueue`则是对`watcher.run`的封装，并添加了对队列排序、重置队列等功能，`watcher.run`方法则会执行`watcher`保存的执行函数，这里就指的是`vm.update(vm.render())`，就是在执行页面的重新渲染。