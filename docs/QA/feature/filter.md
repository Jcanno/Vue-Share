### Q: Vue怎么实现过滤器的？

### A:
过滤器本质是一个函数，如果有这样一个过滤器：
```js
{{ time | formatTime }}
```
会被编译成这样:
```js
_s(_f("formatTime")(time))
```

`_f`指的就是`resolveFilter`，为的是从`this.$options.filters`找到目标过滤器，在这个例子中`resolveFilter`相当于会返回`this.$options.filters['formatTime']`函数，`_f("formatTime")(time)`就是执行了用户编写的过滤器，`_s`就是将过滤器的结果转换成字符串，最终会在渲染阶段将过滤器的结果保存在`vnode.text`中。

