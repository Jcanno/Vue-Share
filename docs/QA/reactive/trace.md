### Q: Vue如何追踪数据的变化？

### A:
Vue中实现响应式编程的核心api是`Object.defineProperty`
```js
function defineReactive (obj, key, val) {
	Object.defineProperty(obj, key, {
		enumerable: true,
    configurable: true,
		get: function reactiveGetter () {
			return val
    },
    set: function reactiveSetter (newVal) {
			val = newVal
    }
	})
}
```

通过简单的封装就可以拦截到对象上属性的获取(`get`)和设置(`set`)，Vue在这两个拦截方法中加入数据响应的功能，就是在`get`中`收集依赖`，在`set`中`触发依赖`，实现视图更新。