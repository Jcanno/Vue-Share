### Q: 整个Diff(虚拟节点比对)过程是怎样的？

### A:
前置概念：
在页面状态发生改变后，Vue会将老虚拟节点(oldVnode)和新虚拟节点(vnode)进行比对，其遍历的算法是vnode与oldVnode同层节点比较，时间复杂度是O(n)，是非常高效的算法比对方式。

Vue会对**元素节点**、**文本节点**、**注释节点**三种节点进行操作并插入到DOM中

比对过程：

1. oldVnode不存在，vnode存在，只需要vnode转换成真实DOM插入到页面中即可，一般发生在首次渲染过程中
2. oldVnode存在，vnode不存在，就需要把oldVnode删除
3. oldVnode和vnode都存在，那就判断两者是否是相同的节点(源码中通过`sameVnode`函数判断)
4. 如果oldVnode和vnode不是相同的节点，那删除掉oldVnode，把vnode插入到DOM中
5. 如果oldVnode和vnode是相同的节点，那就先判断oldVnode和vnode是否都是静态节点，如果是，则跳过diff
6. 如果不是静态节点，接着判断vnode是否是有文本(text)属性，如果是，就将oldVnode的文本改成vnode中的文本
7. 如果vnode不是静态节点并且没有文本属性，那就说明vnode是一个元素节点，元素节点又分为有子节点和无子节点的情况
8. 当vnode不存在子节点时，说明vnode应该是一个空标签，那就删除oldVnode中的子节点或者文本成为一个空标签
9. 当vnode存在子节点时，oldVnode不存在子节点，如果oldVnode还存在text属性，那就清空text，再将vnode的子节点插入
10. 当vnode和oldVnode都存在子节点时，需要将vnode中的vnode与oldVnode遍历对比，Vue在比对children的过程是:
  - 创建oldStartIdx,newStartIdx,oldEndIdx,newEndIdx维护各vnode的下标
  - 通过newStartIdx和oldStartIdx、newStartIdx和oldEndIdx、newEndIdx和oldStartIdx、newEndIdx和oldEndIdx四种特殊情况优化比对，一般情况下子节点的顺序不会有太大的改变，可以使用这种方式快速找到oldVnode中相同的节点
  - 如果以上四种情况都不匹配，那就需要将当前的新子节点与旧子节点一一对比找到相同的节点
  - 遍历结束后，如果还存在新子节点，那就把这些子节点插入父节点中，如果还存在旧子节点，将这些旧子节点删除

