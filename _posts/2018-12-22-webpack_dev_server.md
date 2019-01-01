---
layout: post
title:  "webpack-dev-server"
date:   2018-12-22 15:27:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

**注意attention:** [转载来源webpack 插件拾趣 (1) —— webpack-dev-server](https://www.cnblogs.com/vajoy/p/7000522.html)

webpack-dev-middleware			{#dev_middleware}
====================================

假设我们在服务端使用 express 开发一个站点，同时也想利用 webpack 对静态资源进行打包编译，那么在开发环节，每次修改完文件后，都得先执行一遍 webpack 的编译命令，等待新的文件打包到本地，再做进一步调试。虽然咱们可以利用 webpack 的 watch mode 来监听变更、自动打包，但等待 webpack 重新执行的过程往往很耗时。  

而 webpack-dev-middleware 的出现很好地解决了上述问题 —— 作为一个 webpack 中间件，它会开启 **watch mode** 监听文件变更，并自动地在内存中快速地重新打包、提供新的 bundle。  

说白了就是 —— 自动编译（watch mode）+速度快（全部走内存）！

```javascript
// app.js
const path = require('path');
const express = require("express");
const ejs = require('ejs');
const app = express();
const webpack = require('webpack');
const webpackMiddleware = require("webpack-dev-middleware");
let webpackConf = require('./webpack.config.js');

app.engine('html', ejs.renderFile);
app.set('views', path.join(__dirname, 'src/html'));
app.set("view engine", "html");

var compiler = webpack(webpackConf);

app.use(webpackMiddleware(compiler, {  //使用 webpack-dev-middleware
    publicPath: webpackConf.output.publicPath  //保持和 webpack.config.js 里的 publicPath 一致
}));

app.get("/", function(req, res) {
    res.render("index");
});

app.listen(3333);
```

```javascript
// webpack.config.js
module.exports = {
    entry: './app.js',
    output: {
        publicPath: "/assets/",
        filename: 'bundle.js',
        //path: '/'   //只使用 dev-middleware 可以忽略本属性
```

publicPath —— 它用于决定 webpack 打包编译后的文件，要存放在内存中的哪一个虚拟路径，并提供一个 SERVER，将路径和文件映射起来（即使它们都是虚拟的，但依旧可请求的到）。  

当前的例子，是将内存路径配置为 /assets/，这意味着打包后的 bundle.js 会存放在虚拟内存路径 SERVERROOT/assets/ 下（这里的“SERVERROOT”实际上即 html 文件的访问路径），也意味着我们可以直接在 src/html/index.html 中通过 src='assets/bundle.js' 的形式引用和访问内存中的 bundle 文件:  
```html
<body>
    <div></div>
    <script src="assets/bundle.js"></script>
</body>
```
同时，只要我们修改了页面的脚本模块（比如 src/js/index.js），webpack-dev-middleware 便会自行重新打包到内存，替换掉旧的 bundle，我们只需要刷新页面即可看到刚才的变更。  

HMR			{#HMR}
====================================

HMR 即模块热替换（hot module replacement）的简称，它可以在应用运行的时候，不需要刷新页面，就可以直接替换、增删模块。  

webpack 可以通过配置 webpack.HotModuleReplacementPlugin 插件来开启全局的 HMR 能力，开启后 bundle 文件会变大一些，因为它加入了一个小型的 HMR 运行时（runtime），当你的应用在运行的时候，webpack 监听到文件变更并重新打包模块时，HMR 会判断这些模块是否接受 update，若允许，则发信号通知应用进行热替换。  

webpack-hot-middleware 			{#hot_middleware}
====================================

```javascript
// app.js
app.engine('html', ejs.renderFile);
app.set('views', path.join(__dirname, 'src/html'));
app.set("view engine", "html");

var compiler = webpack(webpackConf);

app.use(webpackMiddleware(compiler, {
    publicPath: webpackConf.output.publicPath
}));

//添加的代码段，引入和使用 webpack-hot-middleware
app.use(require("webpack-hot-middleware")(compiler, {
    path: '/__webpack_hmr'
}));

app.get("/", function(req, res) {
    res.render("index");
});

app.listen(3333);
```

这里的 options 是 webpack-hot-middleware 的配置项，详细见官方文档，这里咱们只填一个必要的 path —— 它表示 webpack-hot-middleware 会在哪个路径生成热更新的事件流服务，且访问的页面会自动与这个路径通过 **EventSource** 进行通讯，来拉取更新的数据重新粉饰自己。  

这里要了解下，实际上 webpack-hot-middleware 最大的能力，是让 SERVER 能够和 HMR 运行时进行通讯，从而对模块进行热更新。  

```javascript
// webpack.config.js
const path = require('path');
const webpack = require('webpack');
module.exports = {
    entry: ['webpack-hot-middleware/client', './app.js'],  //修改点1
    output: {
        publicPath: "/assets/",
        filename: 'bundle.js'
    },
    plugins: [  //修改点2
        new webpack.HotModuleReplacementPlugin(),
        new webpack.NoEmitOnErrorsPlugin()   //出错时只打印错误，但不重新加载页面
    ]
};
```

HTML 5 服务器发送事件               {#server_sent_event}
------------------------------------

```html
<script>
    if(typeof(EventSource)!=="undefined"){
        var source = new EventSource("/example/html5/demo_sse.php");
        source.onmessage = function(event){
            document.getElementById("result").innerHTML+=event.data + "<br />";
        };
    }
    else{
        document.getElementById("result").innerHTML="抱歉，您的浏览器不支持 server-sent 事件 ...";
    }
</script>
```

为了让上面的例子可以运行，您还需要能够发送数据更新的服务器（比如 PHP 和 ASP）。  

服务器端事件流的语法是非常简单的。把 "Content-Type" 报头设置为 "text/event-stream"。现在，您可以开始发送事件流了。  

```php
<?php
header('Content-Type: text/event-stream');
header('Cache-Control: no-cache');

$time = date('r');
echo "data: The server time is: {$time}\n\n";
flush();
?>
```

webpack-dev-server 			{#dev_server}
====================================

顾名思义，webpack-dev-server 相对前两个工具多了个“server”，实际上它的确也是在 webpack-dev-middleware 的基础上多套了一层壳来提供 CLI 及 server 能力  

```javascript
// webpack.config.js
const path = require('path');
const webpack = require('webpack');
module.exports = {
    entry: './app.js',
    output: {
        publicPath: "/assets/",
        filename: 'bundle.js'
    },
    devServer: {
        contentBase: path.join(__dirname, "src/html"),
        port: 3333,
        hot: true  // 让 dev-server 开启 HMR
    },
    plugins: [
        new webpack.HotModuleReplacementPlugin()  //让 webpack 启动全局 HMR
    ]
};
```