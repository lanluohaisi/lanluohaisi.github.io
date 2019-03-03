---
layout: post
title:  "sourceMap学习"
date:   2019-03-03 15:27:01 +0800
categories: other
tag: tip
---

* content
{:toc}

**注意attention:** [学习参考文档：JavaScript Source Map 详解 —— 作者： 阮一峰](http://www.ruanyifeng.com/blog/2013/01/javascript_source_map.html)

**注意attention:** [学习参考文档：Source Map入门教程](https://blog.fundebug.com/2017/03/13/sourcemap-tutorial/)

简单说，Source map就是一个信息文件，里面储存着位置信息。也就是说，转换后的代码的每一个位置，所对应的转换前的位置。  
有了它，出错的时候，除错工具将直接显示原始代码，而不是转换后的代码。这无疑给开发者带来了很大方便。  
目前，暂时只有Chrome浏览器支持这个功能。在Developer Tools的Setting设置中，确认选中"Enable source maps"。  

`如何启用Source map`:  
只要在转换后的代码尾部，加上一行就可以了。  
//@ sourceMappingURL=/path/to/file.js.map  

`如何生成Source map`:  
最常用的方法是使用Google的Closure编译器。生成命令的格式如下：  

```javascript
java -jar compiler.jar \ 
　　　　--js script.js \
　　　　--create_source_map ./script-min.js.map \
　　　　--source_map_format=V3 \
　　　　--js_output_file script-min.js

- js： 转换前的代码文件
　　- create_source_map： 生成的source map文件
　　- source_map_format：source map的版本，目前一律采用V3。
　　- js_output_file： 转换后的代码文件。
```

或者利用前端工具 https://docs.fundebug.com/notifier/javascript/sourcemap/generate/uglify2.html  

`source map 内容`:  

```javascript
// 源码
function o(name) { 
    var o = "Hello, " + name; 
    console.log(o);
}
o('souceMap');

function o(name){var o="Hello, "+name;console.log(o)}o("souceMap");
//# sourceMappingURL=hello.min.js.map

// 生成的Source Map为hello.min.js.map:
{
    "version": 3,
    "sources": [
        "hello.js"
    ],
    "names": [
        "o",
        "name",
        "console",
        "log"
    ],
    "mappings": "AAAA,SAASA,EAAEC,MACP,IAAID,EAAI,UAAYC,KACpBC,QAAQC,IAAIH,GAEhBA,EAAE",
    "sourcesContent": [
        "function o(name) { \n    var o = \"Hello, \" + name; \n    console.log(o);\n}\no('souceMap');"
    ]
}

// version：Source Map的版本号。
// sources：转换前的文件列表。
// names：转换前的所有变量名和属性名。
// mappings：记录位置信息的字符串，经过编码。
// file：(可选)转换后的文件名。
// sourceRoot：(可选)转换前的文件所在的目录。如果与转换前的文件在同一目录，该项为空。
// sourcesContent:(可选)转换前的文件内容列表，与sources列表依次对应。
```

`mappings属性`:  
关键就是map文件的mappings属性。这是一个很长的字符串，它分成三层。  
1. 第一层是行对应，以分号（;）表示，每个分号对应转换后源码的一行。所以，第一个分号前的内容，就对应源码的第一行，以此类推。
2. 第二层是位置对应，以逗号（,）表示，每个逗号对应转换后源码的一个位置。所以，第一个逗号前的内容，就对应该行源码的第一个位置，以此类推。
3. 第三层是位置转换，以VLQ编码表示，代表该位置对应的转换前的源码位置。



