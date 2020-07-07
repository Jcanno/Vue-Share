### Q: 能说说`markStatic`与`markStaticRoots`的区别?

### A:
```js
function markStatic (node: ASTNode) {
  node.static = isStatic(node)
  if (node.type === 1) {
    if (
      !isPlatformReservedTag(node.tag) &&
      node.tag !== 'slot' &&
      node.attrsMap['inline-template'] == null
    ) {
      return
    }
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      if (!child.static) {
        node.static = false
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block
        markStatic(block)
        if (!block.static) {
          node.static = false
        }
      }
    }
  }
}
```
`markStatic`通过`isStatic`给`AST`节点标记静态属性，会遍历当前的子节点，这是`深度优先`的遍历算法，子节点比父节点更早的完成静态标记工作，父节点就可以根据子节点是否是静态节点来决定自身的静态标记属性，若子节点不是静态节点，父节点也不是静态节点，若子节点是静态节点，父节点的静态属性由`isStatic`方法决定。

```js
function markStaticRoots (node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
    if (node.static || node.once) {
      node.staticInFor = isInFor
    }
		// For a node to qualify as a static root, it should have children that
    // are not just static text. Otherwise the cost of hoisting out will
    // outweigh the benefits and it's better off to just always render it fresh.
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
}
```

`markStaticRoots`的作用是标记静态根节点，需要符合这几种情况:
1. 当前节点是静态节点
2. 存在子节点
3. 不能是`只有一个子节点并且是文本节点`的情况

在看看整个`optimize`方法

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

根据上面的代码可以看到，`markStatic`将每个节点标记后，`markStaticRoots`会根据`markStatic`标记的`static`属性进行`staticRoot`标记，在后面更新diff的时候也是根据`staticRoot`属性做更新优化的工作，这也就是说`markStatic`是为`markStaticRoots`做了铺垫的工作。


### Q：为什么`markStaticRoots`判断静态根节点时会将`只有一个子节点且是文本节点`的情况判断为非静态根节点？

### A：

就像源码注释中说的，`只有一个文本静态节点`优化成本会大于收益。
静态根节点会被解析成vnode，并被加入到`_staticTree`对象缓存中，如果只有一个文本节点的节点数量多则占用内存也多，为了减少部分`_staticTree`中占用的内存，就将`只有一个子节点且是文本节点`的节点标记为非静态根节点。