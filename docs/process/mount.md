## 实例的挂载
Vue在实例化调用`_init`方法的最后一个动作是调用`$mount`来实现实例挂载。`$mount`方法定义在`src/platforms/web/runtime/index.js`中，因为我们分析的带有编译器版本的Vue，它在`src/platforms/entry-runtime-with-compiler.js`中在原`$mount`方法的基础上重新定义了`$mount`方法。这里我将这两个`$mount`方法结合在一起方便我们观察。
```js
// src/platforms/web/runtime/index.js
// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

// src/platforms/entry-runtime-with-compiler.js
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

	const options = this.$options

  // resolve template/el and convert to render function
  if (!options.render) {
		let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}
```
这里分为以下几点核心:
- 原`$mount`其实是在调用`mountComponent`方法
- 重新定义的`$mount`方法先对原`$mount`进行保存，若配置中没有`render`函数，则通过相关规则获取`template`并配合`compileToFunctions`方法生成`render`函数。`compileToFunctions`的作用是将`template`编译成`render`函数，有关编译的内容可点击[这里](/compiler/parse.md)
- 最终调用了保存的原`$mount`方法