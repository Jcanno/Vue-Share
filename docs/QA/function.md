### Q: 为何Vue使用`Function`而不是`class`实现?

**A: 有人说Vue使用`Fuction`的原因在于可以将各模块的逻辑分散出去，用`class`难以扩展。但在我看来，Vue同样可以使用`class`实现**。

```js
class Vue {
	constructor(options) {
		this._init(options)
	}
}

initMixin(Vue)
// ...


function initMixin(Vue) {
	Vue.prototype._init = function(options) {
		// ...
	}
}
```
如果不是扩展的原因，那我认为应该是为了兼容`IE`浏览器，使用`Function`实现则可以兼容到`IE 9`，若使用`class`方法实现，最高版本的`IE 11`也无法兼容