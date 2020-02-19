# 项目结构
我们在github上克隆的代码目录如下：
```js
.
├── BACKERS.md
├── LICENSE
├── README.md
├── benchmarks
├── dist
├── examples
├── flow
├── node_modules
├── package-lock.json
├── package.json
├── packages
├── scripts
├── src
├── test
├── types
└── yarn.lock
```
这里挑几个重要的文件夹解释一下：
- **`flow`** 这里放置了`Vue项目中重要变量的类型声明`文件。flow是facebook开发的用于检查JS类型的工具。JS是门动态性极强的语言，许多错误都在运行时才检测出来，有统计分析，50%的前端错误是由于`Type Error`引起的。由此可见，编写JS代码是非常高效的，但会有一定的变量类型隐患，有些隐患甚至会引起程序崩溃。引入类型变量检查虽降低了程序开发的效率，但它使程序更稳定。
- **`types`** 同样是代码类型检查工具，里面放置了ts文件，这是Vue新版引入的功能，支持`TypeScript`。
- **`scripts`** 这里放置了项目需要的脚本构建文件，主要是通过`Rollup`根据不同目标打包项目源码的脚本。
- **`example`** 这里放置了编写Vue代码案例，我们可以仿照它自己编写案例来调试Vue源码。
- **`src`** 这个文件夹是我们`关注的核心`，我们将此展开分析。
  
# src文件夹
```js
.
├── compiler
├── core
├── platforms
├── server
├── sfc
└── shared
```
- **`compiler`** 这个文件夹与Vue编译有关，主要提供将Vue代码转换成`ast树`、再生成`render`函数的功能。如果在运行时执行编译代码，这会`降低程序的运行效率`，所以我们在开发时通常搭配`webpack`来进行预编译。
- **`platforms`** Vue是一个跨平台的框架，它能运行在`web`和`weex`平台上。
- **`server`** 这里提供Vue服务端渲染的功能。
- **`sfc`** sfc(single file components)能将`.vue`单文件解析成JS文件
- **`shared`** 这里放置一些工具方法，主要`web`和`weex`两个平台共有的方法。
- **`core`** 顾名思义，这是scr文件夹的核心，也就是整个项目核心的核心，它包含了Vue整个运行的流程、全局、实例API的定义等。我们再来看`core`中的目录结构。

# core文件夹
```js
.
├── components
├── config.js
├── global-api
├── index.js
├── instance
├── observer
├── util
└── vdom
```
- **`components`** Vue具有组件化开发的功能，Vue内置了多个组件，主要有`Keep-Alive`、`Transition`和`Transition Group`等。这个文件夹只包含了`Keep-Alive`组件，`Transition`和`Transition Group`组件定义在`src/platforms/web/runtime/components`中，这个web平台独有的。
- **`config.js`** Vue项目需用的相关配置。
- **`global-api`** 放置了Vue的全局api，如`Vue.extend`、`Vue.use`等。
- **`instance`** 这里放置了有关Vue实例的运行流程代码，包括组件的渲染、虚拟dom的转换、各个生命周期的声明和调用等。
- **`observer`** 提供有关Vue响应式编程功能的实现方法。
- **`util`** 这个文件夹提供了一系列的工具方法。
- **`vdom`** 这里提供了虚拟dom转换成真实dom、生成虚拟node等方法

