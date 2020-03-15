---
layout: post
title:  "webpack4源码学习(1)--webpack-cli"
date:   2020-01-21 20:02:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

[学习参考:webpack 4 源码主流程分析（二）：配置初始化](https://juejin.im/post/5e1c939ef265da3df07c43aa)  
[学习参考:webpack4 源码分析(二) webpack-cli](https://blog.csdn.net/hjb2722404/article/details/89440492)  
[webpack中文文档](https://www.webpackjs.com/guides/)  
[学习参考：QH-Jimmy 随笔分类 - webpack源码系列](https://www.cnblogs.com/QH-Jimmy/category/1129698.html)

1. 根据 npm 的规则，cli 执行 webpack 后，就会去执行 node_modules/.bin/webpack 文件即 node_modules/webpack/bin/webpack.js  

```js
// 代码截取
const isInstalled = packageName => {
    try {
      require.resolve(packageName); // 使用内部的 require() 机制查询模块的位置，此操作只返回解析后的文件名，不会加载该模块
      return true;
    } catch (err) {
      return false;
    }
};
// 调用require.resolve，然后通过installedClis，判断是否安装了包 webpack-cli 或者 webpack-command
const installedClis = CLIs.filter(cli => cli.installed);

// 未安装则询问安装
const questionInterface = readLine.createInterface({
    input: process.stdin,
    output: process.stderr
});
questionInterface.question(question, answer => {
    questionInterface.close();
    // ···
    // 里面通过 require(packageName) --- 执行 node_modules/webpack-cli/bin/cli.js
});
```

2. webpack-cli/bin/cli.js --立即执行函数，使其可以使用return； 引入 `import-local` 包用于优先选用本地包，`v8-compile-cache` 包用于 v8[缓存优化](https://github.com/flyyang/blog/issues/13)  

```js
// 代码截取
const yargs = require("yargs")
 // 配置了 yargs的帮助等信息
require("./config/config-yargs")(yargs);
yargs.parse(process.argv.slice(2), (err, argv, output) => {
    Error.stackTraceLimit = 30;
    // ...
  
    let options;
    try {
      // 对配置文件和参数进行转换与合法性检测并生成最终的配置选项参数（options）
      options = require("./utils/convert-argv")(argv);
    } catch (err) {
      // ...
      process.exitCode = 1;
      return;
    }

    function processOptions(options) {
      // ...参数options处理等
      //引入webpack包
      const webpack = require("webpack");

      let lastHash = null;
      let compiler;
      try {
        compiler = webpack(options);
      } catch (err) {

      }

      if (argv.progress) {
        const ProgressPlugin = require("webpack").ProgressPlugin;
        new ProgressPlugin({
          profile: argv.profile
        }).apply(compiler);
      }
      // 如果设置了全部输出，则根据参数中是否包含w（是否为监听模式）参数来决定使用哪个方法来显示编译信息
      if (outputOptions.infoVerbosity === "verbose") {
        if (argv.w) {
          compiler.hooks.watchRun.tap("WebpackInfo", compilation => {
            const compilationName = compilation.name ? compilation.name : "";
            console.error("\nCompilation " + compilationName + " starting…\n");
          });
        } else {
          compiler.hooks.beforeRun.tap("WebpackInfo", compilation => {
            const compilationName = compilation.name ? compilation.name : "";
            console.error("\nCompilation " + compilationName + " starting…\n");
          });
        }
        compiler.hooks.done.tap("WebpackInfo", compilation => {
          const compilationName = compilation.name ? compilation.name : "";
          console.error("\nCompilation " + compilationName + " finished\n");
        });
      }
      //编译回调
      function compilerCallback(err, stats) {
        if (!options.watch || err) {
          // Do not keep cache anymore
          //如果不处于监听模式或者编译出错，则净化文件系统，即不再缓存文件
          compiler.purgeInputFileSystem();
        }
        if (err) {
          lastHash = null;
          console.error(err.stack || err);
          if (err.details) console.error(err.details);
          process.exitCode = 1;
          return;
        }
        if (outputOptions.json) {
          stdout.write(JSON.stringify(stats.toJson(outputOptions), null, 2) + "\n");
        } else if (stats.hash !== lastHash) {
          lastHash = stats.hash;
          if (stats.compilation && stats.compilation.errors.length !== 0) {
            const errors = stats.compilation.errors;
            if (errors[0].name === "EntryModuleNotFoundError") {
              console.error("\n\u001b[1m\u001b[31mInsufficient number of arguments or no entry found.");
              console.error(
                "\u001b[1m\u001b[31mAlternatively, run 'webpack(-cli) --help' for usage info.\u001b[39m\u001b[22m\n"
              );
            }
          }
          const statsString = stats.toString(outputOptions);
          const delimiter = outputOptions.buildDelimiter ? `${outputOptions.buildDelimiter}\n` : "";
          if (statsString) stdout.write(`${statsString}\n${delimiter}`);
        }
        if (!options.watch && stats.hasErrors()) {
          process.exitCode = 2;
        }
      }
      //如果处于监听模式，则获取监听模式的配置项，并在标准输入为“end”时结束执行；
      if (firstOptions.watch || options.watch) {
        const watchOptions =
          firstOptions.watchOptions || options.watchOptions || firstOptions.watch || options.watch || {};
        if (watchOptions.stdin) {
          process.stdin.on("end", function(_) {
            process.exit(); // eslint-disable-line
          });
          process.stdin.resume();
        }
        //以监听模式进行编译，编译按成后执行编译回调
        compiler.watch(watchOptions, compilerCallback);
        if (outputOptions.infoVerbosity !== "none") console.error("\nwebpack is watching the files…\n");
      } else {
        //否则，运行编译模块，并在编译关闭后执行编译回调
        compiler.run((err, stats) => {
          if (compiler.close) {
            compiler.close(err2 => {
              compilerCallback(err || err2, stats);
            });
          } else {
            compilerCallback(err, stats);
          }
        });
      }
    }
    //调用processOptions方法来对options进行处理，然后利用compiler.watch或者compiler.run进行webpack编译
    processOptions(options);
  });
```
