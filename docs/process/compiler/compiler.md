## parse
首先我们编写的模板会进入到`parse`函数里，`parse`函数的作用是将`template`转化成`ast`树。`ast`树(抽象语法树)在`eslint`、`babel`等库中都有应用。`parse`函数定义在`src/compiler/parser/index.js`中。
```js
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
	// ...
	parseHTML(template, {...} )
}
```

**核心要点:**
- `parse`函数最后调用`parseHTML`函数，将模板字符串和编译的配置对象传入，配置对象中包括了配置属性和一些回调处理函数

?> `parseHTML`定义在`src/compiler/parser/html-parser.js`中，这里不贴代码，只讲述大概流程，可用调试观察具体的编译过程

1. 循环html字符串
2. 在`parseStartTag`函数使用正则匹配开始标签和标签内属性，匹配完生成对象返回，调用`advance`函数前进模板索引，之后调用`handleStartTag`来处理开始标签，最后调用`options.start`回调函数生成`ASTElement`
3. 继续使用正则匹配标签内容，同样匹配完记录文本内容并调用`options.chars`给当前`currentParent`元素添加`children`
4. 使用正则匹配结束标签，调用`parseEndTag`处理结束标签，最后调用`options.end`来关闭当前堆栈中的`ASTElement`

我们例子生成的ast树如下:

```js
{
	type: 1
	tag: "div"
	attrsList: []
	attrsMap: {}
	rawAttrsMap: {}
	children: {
		type: 3
		text: "hello world"
		start: 5
		end: 16
	}
	start: 0
	end: 22
	plain: true
}
```
`parseHTML`简单说就是对模板的开始标签、内容和结束标签通过正则获取并生成`ASTElement`。

## optimize
```js
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // first pass: mark all non-static nodes.
  markStatic(root)
  // second pass: mark static roots.
  markStaticRoots(root, false)
}
```
`optimize`对ast树添加静态标识，如果满足条件，则添加`static: true`字段，用于渲染时节约资源

## generate

```js
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
	const code = ast ? genElement(ast, state) : '_c("div")'

  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```
`generate`调用`genElement`生成元素字符串

```js
export function genElement (el: ASTElement, state: CodegenState): string {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre
  }

  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    let code
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData(el, state)
      }

      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
    // module transforms
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
		}
    return code
  }
}
```

`genElement`会根据`AST`不同属性来处理生成最终的字符串


我们例子通过`generate`生成的结果如下:
```js
_c('div',[_v("hello world")])
```
这里的`_c`、`_v`是处理对应元素字符串的函数，用于渲染成`vnode`，分别定义在`src/core/instance/render.js`和`src/core/instance/render-helpers/index.js`中


## 总结
整个编译过程是非常复杂的，有许多的分支逻辑，需要处理许多不用的情况，我们可以根据不同的例子来探究具体场景。