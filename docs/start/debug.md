# 调试Vue

本节主要是提供读者自主探索源码的方式，Vue源码是由[Rollup](https://www.rollupjs.com/)构建的，后者是许多JavaScript类库的构建工具。

要开启Vue源码调试需要以下步骤:
- 在Vue源码根目录的`package.json`文件下，在`dev`的script开启`sourceMap`标识，如下代码所示:
```js
"scripts": {
	"dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev --sourcemap",
}
```
- 使用`npm run dev`,`Rollup`会在`dist`目录下生成`vue.js`和`vue.js.map`文件
- 现在可以将`vue.js`引入你需要的文件中，配合你的Vue代码在浏览器中进行调试
  
除此之外，你也可以通过`webpack`来达到调试Vue的目的，这里就不详细描述。