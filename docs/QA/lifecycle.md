### Q: 单组件从实例化到真实dom的生命周期是怎样的?

**A: 单组件从实例化到渲染成真实dom需要经历`beforeCreate`、`created`、`beforeMounted`、`mounted`**。



### Q: 有一组件A，其包含一子组件B，其渲染生命周期的变化是怎样的？

##### A: 父子组件依次执行`A beforeCreate`、`A created`、`A beforeMounted`、`B beforeCreate`、`B created`、`B beforeMounted`、`B mounted`、`A mounted`。
组件A在初始化执行`vm._update(vm._render(), hydrating)`之前，已经经历了`A beforeCreate`、`A created`、`A beforeMounted`三个生命周期，在`_render`阶段,组件A会渲染子组件B，待子组件B渲染完成后，生成真实dom，触发`B mounted`，最后执行`A mounted`。