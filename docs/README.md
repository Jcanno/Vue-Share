# 写在前面

Vue是前端领域炙手可热的三大框架之一，其`MVVM`的架构大大的提高了我们开发中的生产力，因此深入理解Vue源码将帮助我们`使用Vue做更多的事`并`提高我们代码的架构能力`。	

本文档的目的是在`分享个人学习Vue源码心得`的同时也盼望读者`能够自主的探索、思想Vue及其他源码的精髓。`

目前[Vue-v3.0.0-alpha.4](https://github.com/vuejs/vue-next)已经发布，距离正式版本的上线还有一段时间。本文档分享的Vue源码目标版本为`2.x`，`Vue2.x`同样也是非常优秀的版本，学习`2.x`的代码对学习Vue3.x肯定有帮助，并不过时。

本文档将从`Vue的编译、渲染、更新`流程式的阐述Vue运行的内在机制，同样在`Vue的响应式原理`上提供帮助，并提供Vue中的几大特性(`keep-alive、event`等)的解释。

开源社区里有许多其他优秀的Vue源码解析文档，如[Vue.js技术揭秘](https://ustbhuangyi.github.io/vue-analysis/)、[learnVue](https://github.com/answershuto/learnVue)等，它们给予我在学习Vue源码上非常大的帮助。因此，本文档希望能成为这些优秀文档的补充。

!> 本文档Vue源码版本为`2.6.11`