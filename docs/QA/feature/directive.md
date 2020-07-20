### Q: Vue中自定义指令的原理是怎样的？

### A:
用户编写的指令代码在编译的parse阶段(转成AST)会被保存在AST对象中，在generator阶段生成相应的字符串，然后经过渲染保存在vnode中，可以通过`vnode.data.directives`访问到一个元素绑定的指令集合。Vue在新老节点对比的过程中会触发一系列的钩子函数，指令会监听`create`、`update`、`destory`钩子函数，通过这些钩子函数以及新老节点绑定的指令情况来更新指令。

```js
export default {
  create: updateDirectives,
  update: updateDirectives,
  destroy: function unbindDirectives (vnode: VNodeWithData) {
    updateDirectives(vnode, emptyNode)
  }
}
```
directive导出一个对象，分别监听`create`、`update`、`destory`钩子函数，其中的原理都是在调用`updateDirectives`函数

```js
function updateDirectives (oldVnode: VNodeWithData, vnode: VNodeWithData) {
  if (oldVnode.data.directives || vnode.data.directives) {
    _update(oldVnode, vnode)
  }
}
```
这里只要`oldVnode`或者`vnode`任一存在一项，就执行`_update`函数


```js
function _update (oldVnode, vnode) {
  const isCreate = oldVnode === emptyNode
  const isDestroy = vnode === emptyNode
  const oldDirs = normalizeDirectives(oldVnode.data.directives, oldVnode.context)
  const newDirs = normalizeDirectives(vnode.data.directives, vnode.context)

  const dirsWithInsert = []
  const dirsWithPostpatch = []

  let key, oldDir, dir
  for (key in newDirs) {
    oldDir = oldDirs[key]
    dir = newDirs[key]
    if (!oldDir) {
      // new directive, bind
      callHook(dir, 'bind', vnode, oldVnode)
      if (dir.def && dir.def.inserted) {
        dirsWithInsert.push(dir)
      }
    } else {
      // existing directive, update
      dir.oldValue = oldDir.value
      dir.oldArg = oldDir.arg
      callHook(dir, 'update', vnode, oldVnode)
      if (dir.def && dir.def.componentUpdated) {
        dirsWithPostpatch.push(dir)
      }
    }
  }

  if (dirsWithInsert.length) {
    const callInsert = () => {
      for (let i = 0; i < dirsWithInsert.length; i++) {
        callHook(dirsWithInsert[i], 'inserted', vnode, oldVnode)
      }
    }
    if (isCreate) {
      mergeVNodeHook(vnode, 'insert', callInsert)
    } else {
      callInsert()
    }
  }

  if (dirsWithPostpatch.length) {
    mergeVNodeHook(vnode, 'postpatch', () => {
      for (let i = 0; i < dirsWithPostpatch.length; i++) {
        callHook(dirsWithPostpatch[i], 'componentUpdated', vnode, oldVnode)
      }
    })
  }

  if (!isCreate) {
    for (key in oldDirs) {
      if (!newDirs[key]) {
        // no longer present, unbind
        callHook(oldDirs[key], 'unbind', oldVnode, oldVnode, isDestroy)
      }
    }
  }
}
```

这里流程是:
1. 遍历新节点上指令集合，如果当前新节点的指令在老节点上不存在，说明是一个新指令，就触发自定义指令上的`bind`的钩子函数，如果还定义了`inserted`函数，推入`dirsWithPostpatch`数组中
2. 若指令在老节点上存在，就执行自定义指令更新钩子函数的操作，如果还定义了`componentUpdated`函数，推入`dirsWithPostpatch`数组
3. 接着若`dirsWithInsert`有数据，定义一个`callInsert`函数，主要就是遍历`dirsWithInsert`执行`inserted`指令钩子函数，如果当前新节点是刚创建的(通过isCreate判断)，那就把自定义指令的`inserted`方法延迟到vnode插入到真实DOM后再执行。若不是新创建的节点，则直接执行`inserted`钩子函数。
4. `dirsWithPostpatch`和`dirsWithInsert`逻辑类似，将`componentUpdated`钩子函数的执行放置到新老节点比对完成后执行。
5. 最后如果新节点不是刚创建的情况，并且在新节点指令中没有发现老节点指令，说明这个指令被删除了，这时就要执行`unbind`解绑操作。

