---
layout: post
title:  "express和koa学习"
date:   2019-01-19 17:31:01 +0800
categories: js
tag: node
---

* content
{:toc}


express			{#express}
====================================

官网英文  [ http://expressjs.com/](http://expressjs.com/)  
中文  [http://www.expressjs.com.cn/](http://www.expressjs.com.cn/)  
教程[http://www.runoob.com/nodejs/nodejs-express-framework.html](http://www.runoob.com/nodejs/nodejs-express-framework.html)  

入门							{#express_init}
------------------------------------

使用 express-generator 初始化工程  
`npm install express-generator`  
`npx  express --view=ejs myapp`  

```javascript

// myapp/bin/www
#!/usr/bin/env node
/**
 * Module dependencies.
 */
var app = require('../app');
var debug = require('debug')('myapp:server');
var http = require('http');

/**
 * Get port from environment and store in Express.
 */
var port = normalizePort(process.env.PORT || '3000');
app.set('port', port);

/**
 * Create HTTP server.
 * 这里注意，传入的是express的app
 */
var server = http.createServer(app);

/**
 * Listen on provided port, on all network interfaces.
 * 采用的是node的http.Server 类
 */
server.listen(port);
server.on('error', onError);
server.on('listening', onListening);

/**
 * Normalize a port into a number, string, or false.
 */
function normalizePort(val) {
  var port = parseInt(val, 10);

  if (isNaN(port)) {
    // named pipe
    return val;
  }
  if (port >= 0) {
    // port number
    return port;
  }
  return false;
}

/**
 * Event listener for HTTP server "error" event.
 */
function onError(error) {
  if (error.syscall !== 'listen') {
    throw error;
  }

  var bind = typeof port === 'string'
    ? 'Pipe ' + port
    : 'Port ' + port;

  // handle specific listen errors with friendly messages
  switch (error.code) {
    case 'EACCES':
      console.error(bind + ' requires elevated privileges');
      process.exit(1);
      break;
    case 'EADDRINUSE':
      console.error(bind + ' is already in use');
      process.exit(1);
      break;
    default:
      throw error;
  }
}

/**
 * Event listener for HTTP server "listening" event.
 */
function onListening() {
  var addr = server.address();
  var bind = typeof addr === 'string'
    ? 'pipe ' + addr
    : 'port ' + addr.port;
  debug('Listening on ' + bind);
}
```

```javascript
// app.js
var createError = require('http-errors');
var express = require('express');
var path = require('path');
var cookieParser = require('cookie-parser');
var logger = require('morgan');

// 路由
var indexRouter = require('./routes/index');
var usersRouter = require('./routes/users');

var app = express();

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');

app.use(logger('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, 'public')));

app.use('/', indexRouter);
app.use('/users', usersRouter);

// catch 404 and forward to error handler
app.use(function(req, res, next) {
  // next函数主要负责将控制权交给下一个中间件，如果当前中间件没有终结请求，并且next没有被调用，那么请求将被挂起，后边定义的中间件将得不到被执行的机会
  next(createError(404));
});

// error handler
// 中间件，参数个数如果是4个，走error handler，源码中Layer.prototype.handle_error
app.use(function(err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;

```

部分源码							{#express_resource}
------------------------------------

```javascript
// express源码创建app应用部分
/**
 * Create an express application.
 *
 * @return {Function}
 * @api public
 */
function createApplication() {
  // 作为函数传入http.createServer(app)
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };
  // 合并属性merge (dest, src, redefine)，最后一个参数代码是否重新覆盖
  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false);

  // expose the prototype that will get set on requests
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // expose the prototype that will get set on responses
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })
  app.init();
  // 返回app是一个函数
  return app;
}
```

```javascript
// cookie-parser 中间件部分
function cookieParser(secret, options) {
  return function cookieParser(req, res, next) {
    if (req.cookies) {
      return next();
    }

    var cookies = req.headers.cookie;
    var secrets = !secret || Array.isArray(secret)
      ? (secret || [])
      : [secret];

    req.secret = secrets[0];
    req.cookies = Object.create(null);
    req.signedCookies = Object.create(null);

    // no cookies
    if (!cookies) {
      return next();
    }

    req.cookies = cookie.parse(cookies, options);

    // parse signed cookies
    if (secrets.length !== 0) {
      req.signedCookies = signedCookies(req.cookies, secrets);
      req.signedCookies = JSONCookies(req.signedCookies);
    }

    // parse JSON cookies
    req.cookies = JSONCookies(req.cookies);

    next();
  };
}
```

```javascript
// merge-descriptors 源码
// express中的mixin函数 var mixin = require('merge-descriptors');
var hasOwnProperty = Object.prototype.hasOwnProperty
function merge(dest, src, redefine) {
  if (!dest) {
    throw new TypeError('argument dest is required')
  }
  if (!src) {
    throw new TypeError('argument src is required')
  }
  if (redefine === undefined) {
    // Default to true
    redefine = true
  }
  Object.getOwnPropertyNames(src).forEach(function forEachOwnPropertyName(name) {
    if (!redefine && hasOwnProperty.call(dest, name)) {
      // Skip desriptor
      return
    }
    // Copy descriptor
    var descriptor = Object.getOwnPropertyDescriptor(src, name)
    Object.defineProperty(dest, name, descriptor)
  })

  return dest
}
```

```javascript
// https://developer.mozilla.org hasOwnProperty
var foo = {
    hasOwnProperty: function() {
        return false;
    },
    bar: 'Here be dragons'
};
foo.hasOwnProperty('bar'); // 始终返回 false

// 如果担心这种情况，可以直接使用原型链上真正的 hasOwnProperty 方法
({}).hasOwnProperty.call(foo, 'bar'); // true

// 也可以使用 Object 原型上的 hasOwnProperty 属性
Object.prototype.hasOwnProperty.call(foo, 'bar'); // true
```

mysql							{#express_mysql}
------------------------------------

github [https://github.com/mysqljs/mysql](https://github.com/mysqljs/mysql)  

```javascript

var mysql = require('mysql');
// 连接池单例
var poolMap={};
function setDbConfig(conf){
    var conf = conf||dbconfig;
    var poolKey = conf.host+"_"+conf.database;
    if(poolMap[poolKey]){
        return poolMap[poolKey];
    }
    poolMap[poolKey]=mysql.createPool(conf);
    return poolMap[poolKey];
};

var pool = setDbConfig(conf.hm);

pool.getConnection(function(err, connection) {
  if (err) throw err; // not connected!

  // Use the connection
  connection.query('SELECT something FROM sometable', function (error, results, fields) {
    // When done with the connection, release it.
    connection.release();

    // Handle error after the release.
    if (error) throw error;

    // Don't use the connection here, it has been returned to the pool.
  });
});

```


koa			{#koa}
====================================

官网英文  [ https://koajs.com/](https://koajs.com/)  
中文  [https://koa.bootcss.com/](https://koa.bootcss.com/)  
github  [https://github.com/koajs/koa](https://github.com/koajs/koa)  
guthub中文文档  [https://demopark.github.io/koa-docs-Zh-CN/](https://demopark.github.io/koa-docs-Zh-CN/)  

入门						{#koa_init}
------------------------------------

```javascript
// hello world
const Koa = require('koa');
const app = new Koa();

// logger
app.use(async (ctx, next) => {
    console.log('log_start');
    await next();
    console.log('log_end');
    const rt = ctx.response.get('X-Response-Time');
    console.log(`${ctx.method} ${ctx.url} - ${rt}`);
});

// x-response-time
app.use(async (ctx, next) => {
    console.log('x-response-time_start');
    const start = Date.now();
    await next();
    console.log('x-response-time_start_end');
    const ms = Date.now() - start;
    ctx.set('X-Response-Time', `${ms}ms`);
});

// response
app.use(async ctx => {
    console.log('response');
    ctx.body = 'Hello World';
    console.log('response_end');
});

app.listen(3000);

/**
    ET / - 2ms
    log_start
    x-response-time_start
    response
    response_end
    x-response-time_start_end
    log_end
 */

```

中间件middleware            {#koa_middleware}
------------------------------------

Koa 中间件是简单的函数，它返回一个带有签名 (ctx, next) 的MiddlewareFunction。当中间件运行时，它必须手动调用 next() 来运行 “下游” 中间件。  

当创建公共中间件时，将中间件包装在接受参数的函数中，遵循这个约定是有用的，允许用户扩展功能。即使您的中间件 不 接受任何参数，这仍然是保持统一的好方法。  

```javascript
function logger(format) {
  format = format || ':method ":url"';

  return async function (ctx, next) {
    const str = format
      .replace(':method', ctx.method)
      .replace(':url', ctx.url);

    console.log(str);

    await next();
  };
}
app.use(logger());
app.use(logger(':method :url'));

```

部分源码						{#koa_resource}
------------------------------------

```javascript
// application.js
module.exports = class Application extends Emitter {
  /**
   * Initialize a new `Application`.
   *
   * @api public
   */

  constructor() {
    super();

    this.proxy = false;
    this.middleware = [];
    this.subdomainOffset = 2;
    this.env = process.env.NODE_ENV || 'development';
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
    // util.inspect(object,[showHidden],[depth],[colors])是一个将任意对象转换为字符串的方法，通常用于调试和错误输出
    // 对象可以定义自己的 [util.inspect.custom](depth, opts)函数，util.inspect() 会调用并使用查看对象时的结果：
    if (util.inspect.custom) {
      this[util.inspect.custom] = this.inspect;
    }
  }

  /**
   * Shorthand for:
   *
   *    http.createServer(app.callback()).listen(...)
   *
   * @param {Mixed} ...
   * @return {Server}
   * @api public
   */

  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback());
    return server.listen(...args);
  }

  /**
   * Return JSON representation.
   * We only bother showing settings.
   * only 模块是刷新对象中的属性
   * @return {Object}
   * @api public
   */

  toJSON() {
    return only(this, [
      'subdomainOffset',
      'proxy',
      'env'
    ]);
  }

  /**
   * Inspect implementation.
   *
   * @return {Object}
   * @api public
   */

  inspect() {
    return this.toJSON();
  }

  /**
   * Use the given middleware `fn`.
   *
   * Old-style middleware will be converted.
   *
   * @param {Function} fn
   * @return {Application} self
   * @api public
   */

  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }

  /**
   * Return a request handler callback
   * for node's native http server.
   *
   * @return {Function}
   * @api public
   */

  callback() {
    // koa-compose中间件
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }

  /**
   * Handle request in callback.
   *
   * @api private
   */

  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    // 执行compose返回的中间件方法，注意第一次是没有next的方法的
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }

  /**
   * Initialize a new context.
   *
   * @api private
   */

  createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
  }

  /**
   * Default error handler.
   *
   * @param {Error} err
   * @api private
   */

  onerror(err) {
    if (!(err instanceof Error)) throw new TypeError(util.format('non-error thrown: %j', err));

    if (404 == err.status || err.expose) return;
    if (this.silent) return;

    const msg = err.stack || err.toString();
    console.error();
    console.error(msg.replace(/^/gm, '  '));
    console.error();
  }
};

/**
 * Response helper.
 */

function respond(ctx) {
  // allow bypassing koa
  if (false === ctx.respond) return;

  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if ('HEAD' == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }

  // status body
  if (null == body) {
    if (ctx.req.httpVersionMajor >= 2) {
      body = String(code);
    } else {
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      ctx.type = 'text';
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  if (Buffer.isBuffer(body)) return res.end(body);
  if ('string' == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}

```

```javascript
// koa-compose中间件
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */
  // 重点--中间件的dispatch，实现 a -> b -> c -> b -> a
  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        // fn 中的next执行，即执行 dispatch，从而实现深度遍历
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

```javascript
// 关于 __defineSetter__  和 __defineGetter__

// Non-standard and deprecated way
var o = {};
o.__defineSetter__('value', function(val) { this.anotherValue = val; });
o.value = 5;
console.log(o.value); // undefined
console.log(o.anotherValue); // 5


// Standard-compliant ways

// Using the set operator 使用 对象定义时的 set
var o = { set value(val) { this.anotherValue = val; } };
o.value = 5;
console.log(o.value); // undefined
console.log(o.anotherValue); // 5

// Using Object.defineProperty
var o = {};
Object.defineProperty(o, 'value', {
  set: function(val) {
    this.anotherValue = val;
  }
});
o.value = 5;
console.log(o.value); // undefined
console.log(o.anotherValue); // 5


// Non-standard and deprecated way
var o = {};
o.__defineGetter__('gimmeFive', function() { return 5; });
console.log(o.gimmeFive); // 5


// Standard-compliant ways

// Using the get operator 使用 对象定义时的get
var o = { get gimmeFive() { return 5; } };
console.log(o.gimmeFive); // 5

// Using Object.defineProperty
var o = {};
Object.defineProperty(o, 'gimmeFive', {
  get: function() {
    return 5;
  }
});
console.log(o.gimmeFive); // 5
```







