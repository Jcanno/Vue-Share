### Q: 什么是依赖？状态和依赖的关系是怎样的？

### A:
1. 首先我们要明白状态，状态就是那些需要`增加响应功能`的数据，例如
```js
export default {
  data() {
    return {
      a: 'hello'
    }
  }
}
```
变量`a`就是一个状态。

理解了状态，那依赖就是使用这个状态的「东西」，可以是用户编写的模板：
```js
<template>
  <div>
    {{ a }}, world
  </div>
</template>
```
也可以是用户编写的`watch`：

```js
this.$watch('a', function(newVal, oldVal) {
  // 变量a变化
})
```

Vue将这些抽象成了一个`watcher`类，就是状态所需要的依赖。

2. 
正如上面所说，一个状态`a`可能被使用在模板中，也可能被使用在`watch`中，那么状态可以持有多个依赖，即`dep`对应多个`watcher`，当状态变化通知所有的依赖进行更新
对于watcher:

```js
this.$watch(function() {
	return this.name + this.age
}, function(newVal, oldVal) {

})
```

`watcher`可以接收一个函数，我们这个函数拥有两个状态，也就是说`name`或者`age`中任一一个变化watch都要更新，因此`watch`也可能订阅多个状态。

结论：状态与依赖是多对多的关系，一个状态会持有多个依赖，一个依赖会订阅多个状态

