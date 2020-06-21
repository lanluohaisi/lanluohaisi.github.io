---
layout: post
title:  "webpack4源码学习(4)--生成module流程"
date:   2020-03-19 20:02:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

[学习参考: 来源于 可鱼不是鱼的 webpack 4 源码主流程分析（六）：构建 module（上）](https://juejin.im/post/5e1c94595188254e35125019)  
[webpack中文文档](https://www.webpackjs.com/guides/)  

1. 继续执行hooks.afterResolve函数回调，执行`new NormalModule`进行module的初始化，然后跳出 factory 函数，执行 factory 函数回调进行依赖缓存后，再跳回到Compilation.js里的`moduleFactory.create`，执行回调  

2. moduleFactory.create回调里执行：  
```js
// Compilation.js
const addModuleResult = this.addModule(module); // 将这个 `module` 保存到全局的 `Compilation.modules` 数组中和 `_modules` 对象中，判断`_modules`是否有该 module 来设置是否已加载的标识
module = addModuleResult.module;

onModule(module); // 如果是入口文件还会将 modules 保存到 `Compilation.entries`

dependency.module = module;
module.addReason(null, dependency); // 添加该 `module` 被哪些模块依赖
// ...
this.buildModule(module, false, null, null, err => {
   //...
});
```

3. `buildModule`方法会执行 NormalModule.js 里面的 `build` 方法, 设置一些属性后，直接调用了 `this.doBuild`,执行`runLoaders`方法，来自模块loader-runner  

4. `loader-runner`模块：[学习参考：可鱼不是鱼](https://juejin.im/post/5e1c94595188254e35125019)  

```js
// runLoaders -> iteratePitchingLoaders（按正序 require 每个 loader） -> loadLoader（对应的 loader 导出的函数赋值到 loaderContext.loader[i].normal、pitch函数，然后执行pitch函数（如果有的话）） -> processResource（转换 buffer 和设置 loaderIndex） -> iterateNormalLoaders（倒序执行所有 loader）-> runSyncOrAsync（同步或者异步执行 loader）

// 这里的递归由于都是尾递归，所以在性能上不会有问题

// 该方法按正序 require 每个 loader
function iteratePitchingLoaders(options, loaderContext, callback) {
  // abort after last loader 读取所有 loader 后，执行 processResource 方法
  if (loaderContext.loaderIndex >= loaderContext.loaders.length) return processResource(options, loaderContext, callback);

  var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex]; //选择第一个  loader

  // iterate 增序后递归读取下一个 loader
  if (currentLoaderObject.pitchExecuted) {
    loaderContext.loaderIndex++;
    return iteratePitchingLoaders(options, loaderContext, callback);
  }

  // loadLoader函数定义loadLoader.js里面，主要是通过 var module = require(loader.path) 加载该 loader 模块，把 module 赋值到 loader.normal, pitch 赋值到 loader.pitch
  // 对应的 loader 导出的函数赋值到 loaderContext.loader[].normal
  loadLoader(currentLoaderObject, function(err) {
    if (err) {
      loaderContext.cacheable(false);
      return callback(err);
    }
    var fn = currentLoaderObject.pitch; //loadLoader 里会把 module 赋值到 loader.normal, pitch 赋值到 loader.pitch
    currentLoaderObject.pitchExecuted = true;
    if (!fn) return iteratePitchingLoaders(options, loaderContext, callback);

    // 如果有的话，开始执行 pitch 函数，根据参数情况决定是否继续读取剩下的loader
    runSyncOrAsync(fn, loaderContext, [loaderContext.remainingRequest, loaderContext.previousRequest, (currentLoaderObject.data = {})], function(err) {
      if (err) return callback(err);
      var args = Array.prototype.slice.call(arguments, 1);
      if (args.length > 0) {
        loaderContext.loaderIndex--;
        iterateNormalLoaders(options, loaderContext, args, callback);
      } else {
        iteratePitchingLoaders(options, loaderContext, callback);
      }
    });
  });
}

// 转换 buffer 和设置 loaderIndex
function processResource(options, loaderContext, callback) {
  // set loader index to last loader 获取最后一个 loader 的 index
  loaderContext.loaderIndex = loaderContext.loaders.length - 1;

  var resourcePath = loaderContext.resourcePath;
  if (resourcePath) {
    loaderContext.addDependency(resourcePath);
    // 转换为 buffer
    options.readResource(resourcePath, function(err, buffer) {
      if (err) return callback(err);
      options.resourceBuffer = buffer; //得到buffer
      iterateNormalLoaders(options, loaderContext, [buffer], callback);
    });
  } else {
    iterateNormalLoaders(options, loaderContext, [null], callback);
  }
}

//倒序执行所有 loader
function iterateNormalLoaders(options, loaderContext, args, callback) {
  if (loaderContext.loaderIndex < 0) return callback(null, args); //执行完所有 loader 后退出，去执行 iteratePitchingLoaders 回调即 runLoaders 的回调

  var currentLoaderObject = loaderContext.loaders[loaderContext.loaderIndex]; //获取对应 loader

  // iterate 减序后递归执行下一个 loader
  if (currentLoaderObject.normalExecuted) {
    loaderContext.loaderIndex--;
    return iterateNormalLoaders(options, loaderContext, args, callback);
  }

  var fn = currentLoaderObject.normal;
  currentLoaderObject.normalExecuted = true;
  if (!fn) {
    return iterateNormalLoaders(options, loaderContext, args, callback);
  }

  convertArgs(args, currentLoaderObject.raw);

  //执行 loader 函数
  runSyncOrAsync(fn, loaderContext, args, function(err) {
    //loader 执行结果的回调
    if (err) return callback(err);

    var args = Array.prototype.slice.call(arguments, 1); // arg:[] 为 loader 转换结果（字符串或者buffer+可能有的sourcemap）
    iterateNormalLoaders(options, loaderContext, args, callback); //递归，并将转换结果一并传入
  });
}

//同步或者异步执行 loader 函数
function runSyncOrAsync(fn, context, args, callback) {
  var isSync = true;
  var isDone = false;
  var isError = false; // internal error
  var reportedError = false;
  //异步处理（调用的话，会改变isSync值，从而可以走异步方法）
  // context.async 是一个闭包函数，它返回的是 innerCallback，而 innerCallback 内部才是真正执行 runSyncOrAsync 的 callback 函数，这个 callback 会进入下一次的 iterateNormalLoaders 逻辑
  context.async = function async() {
    if (isDone) {
      if (reportedError) return; // ignore
      throw new Error('async(): The callback was already called.');
    }
    isSync = false;
    return innerCallback;
  };
  // 异步后会执行此方法，loader 的结果会作为参数传导出来
  var innerCallback = (context.callback = function() {
    if (isDone) {
      if (reportedError) return; // ignore
      throw new Error('callback(): The callback was already called.');
    }
    isDone = true;
    isSync = false;
    try {
      callback.apply(null, arguments); // arguments 为 loader 结果，第一个值为 null 第二个为字符串或者 buffer，第三个为 SourceMap
    } catch (e) {
      //...
    }
  });
  try {
    var result = (function LOADER_EXECUTION() {
      return fn.apply(context, args); //执行 loader 函数，参数传递前一个 loader 的执行结果
    })();
    if (isSync) {
      isDone = true;
      if (result === undefined) return callback();
      if (result && typeof result === 'object' && typeof result.then === 'function') {
        return result.then(function(r) {
          callback(null, r);
        }, callback);
      }
      return callback(null, result);
    }
  } catch (e) {
    //...
  }
}

```

5. 回到`doBuild`方法，在回调里执行了 createSource 后,判断 loader 的 result 是否有第三个参数对象并且里面存在 webpackAST 属性，如果有则为 ast 赋值到 _ast 上；  
然后回到 this.doBuild 执行回调，在根据项目配置项判断是否需要 parse 后，若需要解析，则执行`this.parser.parse`  
```js
// NormalModule.js 里面的 `build` 方法里
// ...
const result = this.parser.parse(
  this._ast || this._source.source(),
  {
    current: this,
    module: this,
    compilation: compilation,
    options: options
  },
  (err, result) => {
    //...
  }
);
if (result !== undefined) {
  // parse is sync
  handleParseResult(result); // 里面_initBuildHash方法，生成 buildHash
}

// Parser.js 里面的parse方法，里面再执行静态方法 Parser.parse
parse(source, initialState) {
    let ast;
    let comments;
    if (typeof source === "object" && source !== null) {
        ast = source;
        comments = source.comments;
    } else {
        comments = [];
        ast = Parser.parse(source, {
            sourceType: this.sourceType,
            onComment: comments
        });
    }
    // ...

    // 下面代码会根据 import/export 的不同情况即模块间的相互依赖关系，在对应的 module.dependencies 上增加相应的依赖
    // 调用hooks.program.call，触发HarmonyDetectionParserPlugin 和 UseStrictPlugin，增加依赖：HarmonyCompatibilityDependency, HarmonyInitDependency，ConstDependency
    if (this.hooks.program.call(ast, comments) === undefined) {
        this.detectStrictMode(ast.body); // 检测当前执行块是否有 use strict
        // 处理 import 进来的变量，是 import 就增加依赖 HarmonyImportSideEffectDependency，HarmonyImportSpecifierDependency;处理 export 出去的变量，是 export 增加依赖 HarmonyExportHeaderDependency，HarmonyExportSpecifierDependency;还会处理其他相关导入导出的变量
        this.prewalkStatements(ast.body); // 处理块遍历
        this.blockPrewalkStatements(ast.body);
        // 用于深入函数内部（方法在 walkFunctionDeclaration 进行递归），然后递归继续查找 ast 上的依赖，异步此处深入会增加依赖 ImportDependenciesBlock
        this.walkStatements(ast.body);
    }
    this.scope = oldScope;
    this.state = oldState;
    this.comments = oldComments;
    return state;
}
```

6. 回到`moduleFactory.create`回调里的`this.buildModule`,执行`afterBuild`,判断如果该模块是首次解析则执行 processModuleDependencies，里面 对 mudule 的 dependencies, blocks（懒加载 import xx 会存入）, variables（内部变量 __resourceQuery ）分别处理，其中对 blocks 的处理会递归调用。整理过滤没有标识 Identifier 的 module，得到 `sortedDependencies`；  

7. `addModuleDependencies`里，批量调用每个依赖的 NormalModuleFactory.create，即与前文3里面的11点moduleFactory.create 功能一致。所以重复开始走 reslove 流程；  
```js
NormalModuleFactory.create -> resolve流程 -> 初始化module -> add module -> module build -> afterBuild -> processModuleDependencies
```

8. 入口 module 生成: 依赖转换完成后，执行`return process.nextTick(callback);`；将在 nodejs 下一次事件循环时调用 callback 即执行 this.processModuleDependencies 的回调，执行 this._addModuleChain 的回调, 触发 compilation.hooks:succeedEntry，返回入口module，到此 mudule 生成结束!  
```js
// this.processModuleDependencies此时返回一个入口 module
{
  "module": {
    //...
    //同步模块,数组中的每一个Dependency上面都挂载当前所属的module（factory.create里面,dep.module = dependentModule;）
    "dependencies": ["HarmonyImportSideEffectDependency", "HarmonyImportSpecifierDependency"],
    //异步模块
    "blocks": ["ImportDependenciesBlock"]
  }
}
```










