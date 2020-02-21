## 初始化全局API
这节我们来看上节说到的`initGlobalAPI`的方法，定位在`src/core/global-api`中

```js
/* @flow */

import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'
import { observe } from 'core/observer/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  // 2.6 explicit observable API
  Vue.observable = <T>(obj: T): T => {
    observe(obj)
    return obj
  }

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```

`global-api`整个文件夹存放全局api的定义和安装，这里有以下几点核心:
- 在Vue上注册了多个方法和属性，如`set`、`delete`、`nextTick`等
- 以各模块分散安装
	1. initUse:注册`use`方法，用于安装Vue插件
	2. initMixin:注册`mixin`方法，用于混合配置
	3. initExtend:注册`extend`方法，用于组件的继承
	4. initAssetRegisters:注册`components、directive、filter`等方法，用于自定义配置

## 总结
我们通过两节来了解Vue初始化配置，明白Vue在初始化时在原型和全局上注册了一系列的方法，这些方法为Vue的运行打下了基础。