### Q: Vue中computed是如何实现的？

### A:
计算属性是基于`watcher`实现，是一种特殊的`watcher`。当计算属性依赖的状态发生变化，会通知计算属性的`watcher`，`watcher`会将`dirty`设为true，这时会重新计算计算属性的返回值。当`dirty`为false，就说明不需要重新计算返回值，只需要拿到缓存的值即可。

现在版本的Vue会在计算属性依赖的状态改变后，通知到计算属性的`watcher`，由计算属性的`watcher`来检查返回值是否改变，若计算属性的返回值改变，则通知模板的`watcher`或用户的`watcher`进行更新。

```js
const computedWatcherOptions = { lazy: true }

function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

初始化`computed`时，会生成一个`watcher`，`watcher`里的getter函数就是指向我们代理里编写的`computed`，如:

```js
{
	computed: {
		name {
			return this.firstname + this.lastname;
		}
	}
}
```
初始化计算属性的`watcher`后，会检查计算属性是否存在`vm`实例中，若不存在则通过`defineComputed`在`vm`实例上定义当前的computed，若存在则判断与data、props重名，命名重复在开发环境下报警告。 这时计算属性会订阅所有依赖状态，当这些状态改变，计算属性会收到通知，通过计算来决定通知计算属性的依赖。

```js
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

`defineComputed`主要在`vm`添加computed的getter，这个getter不是用户写的computed，而是Vue通过`createComputedGetter`(非SSR环境)包装而成的getter。

```js
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

`createComputedGetter`其实是一个高阶函数，函数中会拿到对应计算属性的`watcher`，当该`watcher`的`dirty`为true，会触发计算属性重新计算，同时如果`Dep.target`存在，将订阅计算属性中所有依赖的状态，好让这些状态改变后能通知到计算属性的`watcher`。

