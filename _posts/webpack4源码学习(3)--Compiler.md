---
layout: post
title:  "webpack4源码学习(3)--Compiler"
date:   2020-03-12 20:02:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

[webpack 4 源码主流程分析（三）：编译前的准备](https://juejin.im/post/5e1c93d6e51d4531220265c9)  
[学习参考:webpack 源码分析（四）——complier模块](https://blog.csdn.net/hjb2722404/article/details/100075212)  
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

7. 













