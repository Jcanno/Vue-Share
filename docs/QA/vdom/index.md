### Q: 为什么index不能作为key？

### A:
给元素设置key是为了在更新时比对新老节点能复用节点做的优化，但使用`v-for`时不能使用`index`作为优化的key。更明确的说，使用`index`作为key并不能起到比对优化的作用。
假如有数据:
```js
[
	{ 
		label: '性别',
		vallue: 'male'
	},
	{ 
		label: '年龄',
		vallue: 25
	},
	{ 
		label: '职业',
		vallue: '教师'
	},
]
```
模板遍历这个数组`info`渲染
``html
<ul v-for=(item, index) in info :key="index">
	<li>
		{{ item.label }}
		{{ item.value }}
	</li>
</ul>
```

我们对数组进行`reserve`反转操作，之后数据为
```js
[
	{ 
		label: '职业',
		vallue: '教师'
	},
	{ 
		label: '年龄',
		vallue: 25
	},
	{ 
		label: '性别',
		vallue: 'male'
	},
]
```
这时会触发Vue的更新节点操作，使用`index`为key时，Vue会认为老数据
```js
{ 
		label: '性别',
		vallue: 'male'
}
```
和新数据
```js
{ 
	label: '职业',
	vallue: '教师'
}
```
对应的节点是相同的节点，但在后面发现他们的子节点中并不相同，所以还是会执行更新的操作，但其实应该拿新数据中最后一项复用即可，这样就没有起到key优化的作用。
正确的做法是给节点设置对应唯一的id，这样就比对时直接复用老节点即可。