* 前言

	* [写在前面](/)

* 开始动手

  * [源码要点](/start/point.md)
  * [调试Vue](/start/debug.md)
  * [项目结构](/start/construction.md)

* Vue运行流程

  * [初始化工作](/process/init.md)

  * **渲染组件**

    * [实例化Vue](/process/renderCom/instance.md)
    * [实例的挂载](/process/renderCom/mount.md)
    * [生成vnode](/process/renderCom/render.md)
    * [执行update](/process/renderCom/update.md)

  * **渲染数据**

    * [初始化组件](/process/renderData/init.md)
    * [渲染数据](/process/renderData/render.md)
    * [生成真实dom](/process/renderData/update.md)

  * **编译**

    * [找到入口](/process/compiler/entry.md)
    * [编译过程](/process/compiler/compiler.md)

  * **响应式编程**

    * [数据响应](/process/dataResponse/data.md)
    * [观察者](/process/dataResponse/observe.md)

* Vue QA

	* [Introduction](/QA/introduction.md)

	* **响应式编程**

		* [Vue如何追踪变化](/QA/reactive/trace.md)
		* [状态与依赖](/QA/reactive/dep.md)
		* [检测数组变化](/QA/reactive/array.md)
		* [set/delete原理](/QA/reactive/set.md)
		* [nextTick的作用](/QA/reactive/nexttick.md)
  
	* **虚拟DOM**

		* [什么是虚拟DOM](/QA/vdom/vdom.md)
		* [Diff过程](/QA/vdom/diff.md)

	* **特性**

		* [自定义指令的原理](/QA/feature/directive.md)
		* [过滤器的原理](/QA/feature/filter.md)
		* [事件方法](/QA/feature/event.md)
		* [父子元素事件绑定](/QA/feature/subevent.md)
		* [实例销毁机制](/QA/feature/destroy.md)

  * [组件的生命周期](/QA/lifecycle.md)
  * [data为何是函数](/QA/data.md)
  
  