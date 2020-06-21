---
layout: post
title:  "webpack4源码学习(5)--生成 chunk及资源构建"
date:   2020-03-30 20:02:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

[学习参考: 来源于 可鱼不是鱼的 webpack 4 源码主流程分析（八）：生成 chunk](https://juejin.im/post/5e1c94abf265da3dfa49ae43)  
[学习参考: 来源于 可鱼不是鱼的 webpack 4 源码主流程分析（一）：前言及总流程概览](https://juejin.im/post/5e1c92776fb9a02fe118628d)  
[webpack中文文档](https://www.webpackjs.com/guides/)  

1. 再上面，处理了entry及依赖的module后，回到了Compile.js 的 compile 的 make 钩子，执行回调`compilation.finish`，finish函数里面，通过compilation.modules，执行`hooks.finishModules.callAsync`,然后回调执行`compilation.seal`，触发了海量 hooks，为我们侵入 webpack 构建流程提供了海量钩子  

2. `hooks.optimizeDependencies`---依赖优化开始时触发；`hooks.beforeChunks`--生成chunk之前  

3. `compilation.seal`函数里面，执行`this.addChunk`，为每一个入口生成一个 chunk，该方法里做了缓存判断后执行 new Chunk(name)，并同时添加 chunk 到 Compilation.chunks  

4. 通过 `buildChunkGraph` 的三个阶段，让所有的 module、chunk、chunkGroup 之间都建立了联系，形成了 chunk Graph, `this.createHash`创建hash；  

5. 在 `createModuleAssets` 里，获取每个 module 属性上的 buildInfo.assets，然后触发 this.emitAsset 生成资源。buildInfo.assets 相关数据可以在 loader 里调用 api: this.emitFile 生成  

6. `createChunkAssets`生成 chunk 资源时，先根据是否含有 runtime 得到不同的 template，包括 chunkTemplate 和 mainTemplate；通过不同的 template 得到不同的 manifest 和 pathAndInfo，然后调用不同的 render `fileManifest.render()`渲染代码；

7. templated的render 会进行`hooks.render.call`, 从而执行当前template的构造函数里面注册的tap，里面执行`hooks.modules.call`, 从而执行`Template.renderChunkModules`, 循环对每一个 module 执行`moduleTemplate.render`, 里面执行`module.source`  
```js
// Template.js
const allModules = modules.map(module => {
    return {
        id: module.id,
        // 循环对每一个 module 执行 render, render 里面会执行module.source
        source: moduleTemplate.render(module, dependencyTemplates, {
            chunk
        })
    };
});
```

8. `module.source` 即执行 NormalModule.js 里的source方法，执行`this.generator.generate`，这个 generator 就是在 reslove 流程 -> getGenerator 所获得，里面 通过 `module.originalSource` 获取 NormalModule的`this._source`(这个_source 是在 doBuild方法里面，执行 `runLoaders`得到的source), 然后执行`this.sourceBlock`，循环处理 module 的每个依赖（module.dependencies）：获得依赖所对应的 template 模板类，然后执行该类的 apply；
>在 apply里，会根据依赖不同做相应的源码转化的处理。但方法里并没有直接执行源码转化的工作，而是将其转化对象 push 到 ReplaceSource.replacements 里;  

9. 无论是同步还是异步，最后都回到 Compilation.js 的 `createChunkAssets` 里，做了 source 缓存，然后执行`this.emitAsset(file, source, assetInfo)`,建立起了文件名与对应源码的联系，将该映射对象挂载到 `compilation.assets` 下。 然后设置了 alreadyWrittenFiles 这个 Map 对象，防止重复构建代码。到此一个 chunk 的资源构建结束  

10. 继续回到`compilation.seal`函数里面，再`this.hooks.afterCompile.callAsync`执行callback函数，回到了最开始的 `run(callback)`, 执行`onCompiled`方法；

11. 执行`this.emitAssets`,先触发`this.hooks.emit.callAsync`, 然后`this.outputFileSystem.mkdirp(outputPath, emitFiles)`, 创建目标文件夹后，执行回调 emitFiles，在回调里通过 asyncLib.forEachLimit 并行执行对每个 file 资源文件进行路径拼接后，将每个 source 源码转换为 buffer 后（性能提升），写入真实路径的 file  
```js
// source 为 CachedSource 实例，source.source 做了缓存判断，执行 this._source.source， this._source 为 ConcatSource 实例，该方法会遍历 children，如果子项不是字符串，则执行其 source 方法。
// 对于 ReplaceSource 实例来说，会执行其 _replaceString 方法，该方法里会循环处理替换在之前 资源的构建 -> 生成 chunk 资源 -> chunkTemplate -> 生成主体 chunk 代码 -> 生成每个 module 代码 push 进去的 replacements，得到替换后的字符串，合并返回 resultStr
let content = source.source(); //source为 CachedSource 实例，content为得到的资源
```