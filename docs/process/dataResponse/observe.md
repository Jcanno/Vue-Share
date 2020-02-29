## observe
```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

**核心要点**
- 实例化`Observe`会初始化`Dep`类，这个类的作用是收集依赖，来保存那些需要响应的数据，在数据更新时来通知数据视图更新
- 缓存`__ob__`观察者对象
- 对需要观察的数据判断，若是数组，则调用`observeArray`，否则调用`walk`

根据我们例子的场景，这里`data`是个对象，则调用`walk`函数，`walk`函数中对对象的key进行遍历，调用`defineReactive`进行数据劫持。

## defineReactive
```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
	const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

**核心要点:**
- 获取属性装饰器`getter`和`setter`
- 对`data`上的值再进行`observe`，这就达到了对数据深度响应更新
- 使用`Object.defineProperty`对属性进行劫持，首先在`get`方法中，会判断`Dep.target`，这个是在实例化`Watcher`做的工作。

## Watcher

在Vue渲染流程中，`mountComponent`方法会实例化`watcher`来执行`updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }`函数，我们先来看`Wathcer`类的get方法
```js
export default class Watcher {
	constructor () {
		// ...
	}

	// ...

	get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    } catch (e) {
      if (this.user) {
        handleError(e, vm, `getter for watcher "${this.expression}"`)
      } else {
        throw e
      }
    } finally {
      // "touch" every property so they are all tracked as
      // dependencies for deep watching
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
}
```

```js
export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}
```
`mountComponent`会实例化`Watcher`执行`get`方法，这里会通过`pushTarget`和`popTarget`方法将`watcher`实例推入或弹出栈。这样就保证了当前只有一个`watcher`来执行数据观察更新的工作。在`pushTarget`后会执行`updateComponent`即`vm._update(vm._render(), hydrating)`，在`_render`函数中Vue会对`render`函数的数据进行获取，这就触发了数据`getter`方法，


## Dep
`Dep.target`是`Dep Watcher类型`的静态属性，`pushTarget`会将`watcher`实例推入堆栈中，回到`defineReactive`函数`Object.defineProperty`中的`get`方法，判断`Dep.target`的值，此时为`true`，则会执行`dep.depend()`方法。

```js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

执行`depend`方法调用`Dep.target`即`watcher`实例的`addDep`方法

```js
export default class Watcher {
	// ...
	constructor () {
		// ...
	}

	addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }
}
```
**核心要点:**
- `watcher`实例会将`dep`推入到`newDeps`中，`watcher`就持有了`dep`，`watcher`有`deps`和`newDeps`属性，又执行dep的`addSub`方法。

```js
addSub (sub: Watcher) {
  this.subs.push(sub)
}
```
这时`dep`也持有了`watcher`对象

回到`defineReactive`函数的`set`方法中，数据若发生了改变，则会调用`notify`方法。

```js
notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  if (process.env.NODE_ENV !== 'production' && !config.async) {
    // subs aren't sorted in scheduler if not running async
    // we need to sort them now to make sure they fire in correct
    // order
    subs.sort((a, b) => a.id - b.id)
  }
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```
这时会遍历该`dep`持有的`watcher`数组，遍历数组执行`update`方法。`update`则是在执行对视图更新工作

## 总结
`observe`对数据实现观察更新，通过`Dep`类对数据进行依赖收集，这里Vue就知道了哪些数据需要观察更新。在数据发生改变的时候，通过`dep.notify`方法通知`watcher`更新视图。