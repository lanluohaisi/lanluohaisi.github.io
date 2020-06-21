---
layout: post
title:  "webpack4源码学习(3)--Compiler与Compilation与resolver流程"
date:   2020-03-12 20:02:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

[webpack 4 源码主流程分析（三）：编译前的准备](https://juejin.im/post/5e1c93d6e51d4531220265c9)  
[学习参考:webpack 源码分析（四）——complier模块](https://blog.csdn.net/hjb2722404/article/details/100075212)  
[学习参考：QH-Jimmy 随笔分类 - webpack源码系列](https://www.cnblogs.com/QH-Jimmy/category/1129698.html)
[webpack中文文档](https://www.webpackjs.com/guides/)

1. validateSchema方法，使用`ajv` 以及 `ajv-keywords`，校验options对象  
[文档链接 https://ajv.js.org/](https://ajv.js.org/)  
```js
// Node.js require:
var Ajv = require('ajv');
// or ESM/TypeScript import
import Ajv from 'ajv';

var ajv = new Ajv(); // options can be passed, e.g. {allErrors: true}
var validate = ajv.compile(schema);
var valid = validate(data);
if (!valid) console.log(validate.errors);

// webpack相关代码
const Ajv = require("ajv");
const ajv = new Ajv({
	errorDataPath: "configuration",
	allErrors: true,
	verbose: true
});
require("ajv-keywords")(ajv, ["instanceof"]);
require("../schemas/ajv.absolutePath")(ajv);

const validateObject = (schema, options) => {
    const validate = ajv.compile(schema); //校验
    const valid = validate(options);
    return valid ? [] : filterErrors(validate.errors);
};
```

2. `options = new WebpackOptionsDefaulter().process(options);`--合并默认配置项目配置  
`compiler = new Compiler(options.context);`--context为当前项目绝对路径

3. `new Compiler` 执行 constructor，扩展了 Tapable，在 constructor 里定义了一堆钩子 done,beforeRun,run,emit 等等;  
`this._pluginCompat.tap("Compiler", options => {...});` 来兼容之前的老版 webpack 的 plugin 的钩子，触发时机在tapable/lib/Tapable.js里调用plugin 的时候，利用_pluginCompat设置hooks的注册；  
`this.resolverFactory = new ResolverFactory()`，利用HookMap来进行hooks的注册，在plugin方法时，执行`this._pluginCompat.tap("ResolverFactory", options => {...})`,利用正则匹配，注册resolver的hooks；可参考[enhanced-resolve](https://github.com/webpack/enhanced-resolve)  
`this.requestShortener = new RequestShortener(context);`--用于缩短路径？？  

4. `new NodeEnvironmentPlugin({infrastructureLogging: options.infrastructureLogging}).apply(compiler)`, compiler对象扩展文件属性，inputFileSystem、outputFileSystem、watchFileSystem，这些属性上利用graceful-fs以及fs挂载node的fs文件系统，比如`inputFileSystem.readdir、outputFileSystem.writeFile`等；利用`watchpack` 模块对input文件进行文件监听；注册compiler.hooks.beforeRun事件流，清除storage；  

5. 执行配置的options.plugins，执行apply方法，然后调用environment、afterEnvironment的hook的call方法，执行注册的tap；有callback回调的话，执行 `compiler.watch`--有watch配置 或者 `compiler.run`--有callback回调；返回compiler对象(注意没有callback的话，只是返回对象，并不执行run或者watch)  
```js
for (const plugin of options.plugins) {
    if (typeof plugin === "function") {
        plugin.call(compiler, compiler);
    } else {
        plugin.apply(compiler); // 传入compiler
    }
}
```

6. `compiler.options = new WebpackOptionsApply().process(options, compiler);`--加载了大量的插件，触发compiler.hooks.entryOption、afterPlugins、afterResolvers事件流，具体学习参考 [浅析webpack源码之WebpackOptionsApply模块-plugin事件流总览](https://www.cnblogs.com/QH-Jimmy/p/8081069.html)  

7. `Compiler的 run(callback)`--this.hooks.beforeRun.callAsync、run.callAsync的调用,然后在它的回调里执行 this.compile(onCompiled)  

8. `Compiler的 compile(onCompiled)`--是真正进行编译的过程，最终会把所有原始资源编译为目标资源。实例化了一个 compilation，并将 compilation 传给 make 钩子上的方法，注册在这些钩子上的方法会调用 compilation 上的 addEntry，执行构建  
```js
// Compiler.js
const params = this.newCompilationParams(); // 实例化了 NormalModuleFactory 类和 ContextModuleFactory 类

// NormalModuleFactory.js
this.ruleSet = new RuleSet(options.defaultRules.concat(options.rules)); // 规范化配置的 module.rules

// Compiler.js
// 实例化了一个 Compilation，也是扩展于 tapable。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息，代表了一次资源的构建
const compilation = this.newCompilation(params);

// 模板解析的辅助模块
this.mainTemplate = new MainTemplate(this.outputOptions); // 输出主模板对象
this.chunkTemplate = new ChunkTemplate(this.outputOptions);
// ...

// 调用之前WebpackOptionsApply里面注册的大量插件的thisCompilation、compilation函数钩子，然后函数里面，也是注册事件，包括compilation、normalModuleFactory、contextModuleFactory等，还有里的子对象mainTemplate等
this.hooks.thisCompilation.call(compilation, params);
this.hooks.compilation.call(compilation, params);

// hooks.make的主要来源在EntryOptionPlugin插件中，无论entry参数是单入口字符串、单入口数组、多入口对象还是动态函数，都会在引入对应的入口插件后，注入一个make事件。
// 而make事件，会调用compilation.addEntry函数
this.hooks.make.callAsync(compilation, err => {
    if (err) return callback(err);

    compilation.finish(err => {
        if (err) return callback(err);

        compilation.seal(err => {
            if (err) return callback(err);

            this.hooks.afterCompile.callAsync(compilation, err => {
                if (err) return callback(err);

                return callback(null, compilation);
            });
        });
    });
});
```

9. `compilation.addEntry(context, dep, name, callback)` 里面调用 `_addModuleChain`方法，传入回调hooks.succeedEntry.call  
```js
// Compilation.js的_addModuleChain方法
// dependencyFactories包含了所有的依赖集合，是在 compiler.hooks.compilation调用的时候注入的
const Dep = /** @type {DepConstructor} */ (dependency.constructor);
const moduleFactory = this.dependencyFactories.get(Dep); // 获取对应的模块工厂类
this.semaphore.acquire(() => {
    moduleFactory.create(
        // ...
    );
});

//例如 SingleEntryPlugin.js
compiler.hooks.compilation.tap(
    "SingleEntryPlugin",
    (compilation, { normalModuleFactory }) => {
        compilation.dependencyFactories.set(
            SingleEntryDependency,
            normalModuleFactory // 上面moduleFactory get获取的就是normalModuleFactory模块
        );
    }
);
compiler.hooks.make.tapAsync(
    "SingleEntryPlugin",
    (compilation, callback) => {
        const { entry, name, context } = this;

        const dep = SingleEntryPlugin.createDependency(entry, name);
        compilation.addEntry(context, dep, name, callback);
    }
);
```

10. `this.semaphore` --Semaphore.js编译队列控制，原理很简单，对执行进行了并发控制，默认并发数为 100，超过后存入 semaphore.waiters，根据情况再调用 semaphore.release 去执行存入的事件 semaphore.waiters  

11. `normalModuleFactory.js 里面的 moduleFactory.create`-- 将上下文、入口文件、入口模块依赖类整合，然后开始触发normalModuleFactory类上的事件流`beforeResolve、factory、resolver`   
```js
// normalModuleFactory.js
this.hooks.resolver.tap("NormalModuleFactory", () => (data, callback) => {
    // ...
    // getResolver 会执行 resolverFactory.get，判断缓存后，即执行ResolverFactory.js 里的 _create, 然后执行下 
    const loaderResolver = this.getResolver("loader");
    const normalResolver = this.getResolver("normal", data.resolveOptions);
    // ...
    // 过滤 webpack.config.js 中得到 module.rules 所需要的 loader。
    const result = this.ruleSet.exec({
        // ...
    });
    // ...
    callback(null, {
        context: context,
        request: loaders
            .map(loaderToIdent)
            .concat([resource])
            .join("!"),
        dependencies: data.dependencies,
        userRequest,
        rawRequest: request,
        loaders,
        resource,
        matchResource,
        resourceResolveData,
        settings,
        type,
        // 执行 createParser，方法里触发 NormalModuleFactory.hooks:createParser for (type)，该事件注册在 JavascriptModulesPlugin 插件，根据 type 不同返回不同的 parser 实例
        // 实例化之后，触发 NormalModuleFactory.hooks:parser for (type)，会去注册一些在 parser 阶段（遍历解析 ast 的时候）被触发的 hooks。
        parser: this.getParser(type, settings.parser),
        // 执行 createGenerator，方法里触发 NormalModuleFactory.hooks:createGenerator for (type),该事件注册在 JavascriptModulesPlugin 插件，根据 type 不同返回不同的 generator 实例（目前代码里都是返的一致的 new JavascriptGenerator() ）。实例化之后，触发 NormalModuleFactory.hooks:generator for (type)。
        generator: this.getGenerator(type, settings.generator),
        resolveOptions
    });
});

// ResolverFactory.js里的 _create方法里面的下面这行call
// 会执行WebpackOptionsApply里面compiler.resolverFactory.hooks.resolveOptions注册的事件
resolveOptions = this.hooks.resolveOptions.for(type).call(resolveOptions);
const resolver = Factory.createResolver(resolveOptions);
```














