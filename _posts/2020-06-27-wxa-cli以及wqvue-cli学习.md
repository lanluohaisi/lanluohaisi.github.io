---
layout: post
title:  "wxa-cli以及wqvue-cli学习"
date:  2020-06-27 11:02:01 +0800
categories: js
tag: node
---

* content
{:toc}

1. figlet  --- 基于ASCII字符组成的字符画

2. webpack-chain  

3. commander.js [commander github](https://github.com/tj/commander.js/blob/master/Readme_zh-CN.md)  
```js
// 通过绑定处理函数实现命令（这里的指令描述为放在`.command`中）
// 返回新生成的命令（即该子命令）以供继续配置
program
  .command('clone <source> [destination]')
  .description('clone a repository into a newly created directory')
  .action((source, destination) => {
    console.log('clone command called');
  });

// 通过独立的的可执行文件实现命令 (注意这里指令描述是作为`.command`的第二个参数)
// 返回最顶层的命令以供继续添加子命令
// 当.command()带有描述参数时，就意味着使用独立的可执行文件作为子命令。 Commander 将会尝试在入口脚本（例如 ./examples/pm）的目录中搜索program-command形式的可执行文件，例如pm-install, pm-search。通过配置选项executableFile可以自定义名字。

你可以在可执行文件里处理（子）命令的选项，而不必在顶层声明它们。
program
  .command('start <service>', 'start named service')
  .command('stop [service]', 'stop named service, or all if no name supplied');

```

