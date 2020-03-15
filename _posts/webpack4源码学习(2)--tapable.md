---
layout: post
title:  "webpack4源码学习(2)--tapable"
date:   2020-03-08 20:02:01 +0800
categories: js
tag: webpack
---

* content
{:toc}

[github:tapable](https://github.com/webpack/tapable)
[转载学习参考:webpack4核心模块tapable源码解析](https://www.cnblogs.com/tugenhua0707/p/11317557.html#_labe4)  
[学习参考:可能是全网最全最新最细的 webpack-tapable-2.0 的源码分析](https://juejin.im/post/5c12046af265da612b1377aa#heading-13)  



1. tapable 钩子介绍  
```js
// 第一种分类：Sync--同步钩子，只能使用tap方法注册函数；AsyncParallel--异步并行，可以使用tap、tapAsync、tapPromise注册，并行调用；AsyncSeries--可以使用tap、tapAsync、tapPromise注册，异步串行；
// 第二种分类：Basic hook --基本钩子，按顺序简单的调用每一个注册的函数；Waterfall --瀑布流钩子，也是按顺序简单的调用每一个注册的函数，但它将一个返回值从每个函数传递到下一个函数；Bail -- 当任何被注册的函数返回任何内容时，钩子将停止执行其余的钩子；loop--循环钩子
const {
    SyncHook,
    SyncBailHook,
    SyncWaterfallHook,
    SyncLoopHook,
    AsyncParallelHook,
    AsyncParallelBailHook,
    AsyncSeriesHook,
    AsyncSeriesBailHook,
    AsyncSeriesWaterfallHook
} = require("tapable");
```

2. 本质：一种事件监听触发机制，tap、tapAsync、tapPromise进行事件注册，call、promise、callAsync进行事件的触发；不同的钩子，调用的方式不一致；  
精髓：结构特别巧妙，代码复用，缓存的利用，以及new Function生成函数，牛牛的~~  

```js
// github示例
    class Car {
        constructor() {
            // 实例，可选参数数组，定义了参数名列表
            this.hooks = {
                accelerate: new SyncHook(["newSpeed"]),
                brake: new SyncHook(),
                calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
            };
        }
        /* ... */
    }
    const myCar = new Car();
    // 注册，第一个参数options对象，如果传字符串，则options = {name: options};
    // 第二个参数为注册函数，参数为实例时的参数名列表
    myCar.hooks.brake.tap("WarningLampPlugin", () => warningLamp.on());
    myCar.hooks.accelerate.tap("LoggerPlugin", newSpeed => console.log(`Accelerating to ${newSpeed}`));
    myCar.hooks.calculateRoutes.tapPromise("GoogleMapsPlugin", (source, target, routesList) => {
        // return a promise
        return google.maps.findRoute(source, target).then(route => {
            routesList.add(route);
        });
    });
    myCar.hooks.calculateRoutes.tapAsync("BingMapsPlugin", (source, target, routesList, callback) => {
        bing.findRoute(source, target, (err, route) => {
            if(err) return callback(err);
            routesList.add(route);
            // call the callback
            callback();
        });
    });
    myCar.hooks.calculateRoutes.tap("CachedRoutesPlugin", (source, target, routesList) => {
        const cachedRoute = cache.get(source, target);
        if(cachedRoute)
            routesList.add(cachedRoute);
    })
    // 调用
    class Car {
        /**
        * You won't get returned value from SyncHook or AsyncParallelHook,
        * to do that, use SyncWaterfallHook and AsyncSeriesWaterfallHook respectively
        **/
        setSpeed(newSpeed) {
            // following call returns undefined even when you returned values
            this.hooks.accelerate.call(newSpeed);
        }
        useNavigationSystemPromise(source, target) {
            const routesList = new List();
            return this.hooks.calculateRoutes.promise(source, target, routesList).then((res) => {
                // res is undefined for AsyncParallelHook
                return routesList.getRoutes();
            });
        }

        useNavigationSystemAsync(source, target, callback) {
            const routesList = new List();
            this.hooks.calculateRoutes.callAsync(source, target, routesList, err => {
                if(err) return callback(err);
                callback(null, routesList.getRoutes());
            });
        }
    }
```

3. 源码浅析  

+ 因为每个hook结构一样，所从最简单的SyncHook开始  
```js
    // SyncHook.js
    const Hook = require("./Hook");
    const HookCodeFactory = require("./HookCodeFactory");

    class SyncHookCodeFactory extends HookCodeFactory {
         // factory.create方法，在contentWithInterceptors里面，会调用content方法生成call、callAsync、promise函数
        content({ onError, onDone, rethrowIfPossible }) {
            return this.callTapsSeries({
                onError: (i, err) => onError(err),
                onDone,
                rethrowIfPossible
            });
        }
    }

    const factory = new SyncHookCodeFactory();

    const TAP_ASYNC = () => {
        throw new Error("tapAsync is not supported on a SyncHook");
    };
    const TAP_PROMISE = () => {
        throw new Error("tapPromise is not supported on a SyncHook");
    };
     // 生成调用函数的方法，也就是call、callAsync、promise方法，会在内部调用compile方法
    const COMPILE = function(options) {
        factory.setup(this, options);
        return factory.create(options);
    };
    // 定义hook（不理解这种方式和下面的extends方式有什么区别，有什么优势）
    function SyncHook(args = [], name = undefined) {
        const hook = new Hook(args, name);
        hook.constructor = SyncHook;
        hook.tapAsync = TAP_ASYNC;
        hook.tapPromise = TAP_PROMISE;
        hook.compile = COMPILE;
        return hook;
    }
    SyncHook.prototype = null; // 没有理解（难道是因为这样就可以防止改动到Hook原型）

    module.exports = SyncHook;
```

```js
    class SyncHook extends Hook {
        tapAsync() {
            throw new Error("tapAsync is not supported on a SyncHook");
        }
        tapPromise() {
            throw new Error("tapPromise is not supported on a SyncHook");
        }
        compile(options) {
            factory.setup(this, options);
            return factory.create(options);
        }
    }
```

+ 每个hook都是实例化Hook基类  
```js
    const CALL_DELEGATE = function(...args) {
        // 赋值缓存，对于每一个实例，生成一次调用函数后，缓存~~
        this.call = this._createCall("sync");
        return this.call(...args);
    };
    const CALL_ASYNC_DELEGATE = function(...args) {
        this.callAsync = this._createCall("async");
        return this.callAsync(...args);
    };
    const PROMISE_DELEGATE = function(...args) {
        this.promise = this._createCall("promise");
        return this.promise(...args);
    };

    class Hook {
        constructor(args = [], name = undefined) {
            this._args = args;
            this.name = name;
            this.taps = []; // 存储每一个注册对象，包括 name, type, fn 以及其他 options
            this.interceptors = [];
            this._call = CALL_DELEGATE; // 内部，用于重置
            this.call = CALL_DELEGATE;
            this._callAsync = CALL_ASYNC_DELEGATE;
            this.callAsync = CALL_ASYNC_DELEGATE;
            this._promise = PROMISE_DELEGATE;
            this.promise = PROMISE_DELEGATE;
            this._x = undefined; // 用于存储tab注册的函数fn

            this.compile = this.compile; // ???
            this.tap = this.tap;
            this.tapAsync = this.tapAsync;
            this.tapPromise = this.tapPromise;
        }

        compile(options) {
            throw new Error("Abstract: should be overridden");
        }
        // 生成call方法的中间方法
        _createCall(type) {
            return this.compile({ // 每个Hook内部实现的方法，也就是调用factory.create生成
                taps: this.taps,
                interceptors: this.interceptors,
                args: this._args,
                type: type
            });
        }
        // 真正触发的注册事件方法
        _tap(type, options, fn) {
            if (typeof options === "string") {
                options = {
                    name: options
                };
            } else if (typeof options !== "object" || options === null) {
                throw new Error("Invalid tap options");
            }
            if (typeof options.name !== "string" || options.name === "") {
                throw new Error("Missing name for tap");
            }
            if (typeof options.context !== "undefined") {
                deprecateContext();
            }
            options = Object.assign({ type, fn }, options);
            options = this._runRegisterInterceptors(options);
            this._insert(options); // 排序，以及存储到this.taps
        }

        tap(options, fn) {
            this._tap("sync", options, fn);
        }
        tapAsync(options, fn) {
            this._tap("async", options, fn);
        }
        tapPromise(options, fn) {
            this._tap("promise", options, fn);
        }
        // 运行拦截器对options进行过滤
        _runRegisterInterceptors(options) {
            for (const interceptor of this.interceptors) {
                if (interceptor.register) {
                    const newOptions = interceptor.register(options);
                    if (newOptions !== undefined) {
                        options = newOptions;
                    }
                }
            }
            return options;
        }

        withOptions(options) {
            const mergeOptions = opt =>
                Object.assign({}, options, typeof opt === "string" ? { name: opt } : opt);

            return {
                name: this.name,
                tap: (opt, fn) => this.tap(mergeOptions(opt), fn),
                tapAsync: (opt, fn) => this.tapAsync(mergeOptions(opt), fn),
                tapPromise: (opt, fn) => this.tapPromise(mergeOptions(opt), fn),
                intercept: interceptor => this.intercept(interceptor),
                isUsed: () => this.isUsed(),
                withOptions: opt => this.withOptions(mergeOptions(opt))
            };
        }

        isUsed() {
            return this.taps.length > 0 || this.interceptors.length > 0;
        }

        intercept(interceptor) {
            this._resetCompilation();
            this.interceptors.push(Object.assign({}, interceptor));
            if (interceptor.register) {
                for (let i = 0; i < this.taps.length; i++) {
                    this.taps[i] = interceptor.register(this.taps[i]);
                }
            }
        }
        _resetCompilation() { // 重置
            this.call = this._call;
            this.callAsync = this._callAsync;
            this.promise = this._promise;
        }

        _insert(item) {
            // 每次注册都会重置，所以有新的注册，则会重新生成调用函数
            this._resetCompilation();
            let before;
            if (typeof item.before === "string") {
                before = new Set([item.before]);
            } else if (Array.isArray(item.before)) {
                before = new Set(item.before);
            }
            let stage = 0;
            if (typeof item.stage === "number") {
                stage = item.stage;
            }
            let i = this.taps.length;
            while (i > 0) {
                i--;
                const x = this.taps[i]; // i以及i+1都先赋值为i的值
                this.taps[i + 1] = x;
                const xStage = x.stage || 0;
                if (before) { // 查找插入
                    if (before.has(x.name)) {
                        before.delete(x.name);
                        continue;
                    }
                    if (before.size > 0) {
                        continue;
                    }
                }
                if (xStage > stage) {
                    continue;
                }
                i++;
                break;
            }
            this.taps[i] = item;
        }
    }

    Object.setPrototypeOf(Hook.prototype, null);

    module.exports = Hook;
```

+ 每个hook的生成函数，都是实例化HookCodeFactory基类
```js
class HookCodeFactory {
	constructor(config) {
		this.config = config;
		this.options = undefined;
		this._args = undefined;
	}
    /**
     * 根据传入的参数生成call函数的方法
     * @param {{taps: Array<Tap>,interceptors:Array<Interceptor>,args,type}} options
     */
	create(options) {
		this.init(options);
		let fn;
		switch (this.options.type) {
			case "sync":
				fn = new Function(
					this.args(),
					'"use strict";\n' +
						this.header() +
						this.contentWithInterceptors({ // 主打生成函数方法，通过不同的参数onError，onResult，onDone处理调用
							onError: err => `throw ${err};\n`,
							onResult: result => `return ${result};\n`, // 处理返回值的基础代码
							resultReturns: true, // 是否返回结果
							onDone: () => "",
							rethrowIfPossible: true // 是否抛异常
						})
				);
				break;
			case "async":
				fn = new Function(
					this.args({
						after: "_callback" // async类型会多一个参数，callback
					}),
					'"use strict";\n' +
						this.header() +
						this.contentWithInterceptors({
							onError: err => `_callback(${err});\n`,
							onResult: result => `_callback(null, ${result});\n`,
							onDone: () => "_callback();\n"
						})
				);
				break;
			case "promise":
				let errorHelperUsed = false;
				const content = this.contentWithInterceptors({
					onError: err => {
						errorHelperUsed = true;
						return `_error(${err});\n`;
					},
					onResult: result => `_resolve(${result});\n`,
					onDone: () => "_resolve();\n"
				});
				let code = "";
				code += '"use strict";\n';
				code += "return new Promise((_resolve, _reject) => {\n"; // 生成promise函数
				if (errorHelperUsed) {
					code += "var _sync = true;\n";
					code += "function _error(_err) {\n";
					code += "if(_sync)\n";
					code += "_resolve(Promise.resolve().then(() => { throw _err; }));\n";
					code += "else\n";
					code += "_reject(_err);\n";
					code += "};\n";
				}
				code += this.header();
				code += content;
				if (errorHelperUsed) {
					code += "_sync = false;\n";
				}
				code += "});\n";
				fn = new Function(this.args(), code);
				break;
		}
        this.deinit();
        // 哈哈，为了打印生成的方法
        console.log((options.args || []).join(','), fn.toString());
		return fn;
	}

	setup(instance, options) {
		instance._x = options.taps.map(t => t.fn); // 过滤注册函数fn，存储到_x对象里
	}

	/**
	 * @param {{ type: "sync" | "promise" | "async", taps: Array<Tap>, interceptors: Array<Interceptor> }} options
	 */
	init(options) {
		this.options = options;
		this._args = options.args.slice();
	}

	deinit() {
		this.options = undefined;
		this._args = undefined;
	}

	contentWithInterceptors(options) {
		if (this.options.interceptors.length > 0) {
			const onError = options.onError;
			const onResult = options.onResult;
            const onDone = options.onDone;
            // 这里回调用子类hook的content方法
			return this.content(
				Object.assign(options, {
					onError:
						onError &&
						(err => {
							let code = "";
							for (let i = 0; i < this.options.interceptors.length; i++) {
								const interceptor = this.options.interceptors[i];
								if (interceptor.error) {
									code += `${this.getInterceptor(i)}.error(${err});\n`;
								}
							}
							code += onError(err);
							return code;
						}),
					onResult:
						onResult &&
						(result => {
							let code = "";
							for (let i = 0; i < this.options.interceptors.length; i++) {
								const interceptor = this.options.interceptors[i];
								if (interceptor.result) {
									code += `${this.getInterceptor(i)}.result(${result});\n`;
								}
							}
							code += onResult(result);
							return code;
						}),
					onDone:
						onDone &&
						(() => {
							let code = "";
							for (let i = 0; i < this.options.interceptors.length; i++) {
								const interceptor = this.options.interceptors[i];
								if (interceptor.done) {
									code += `${this.getInterceptor(i)}.done();\n`;
								}
							}
							code += onDone();
							return code;
						})
				})
			);
		} else {
			return this.content(options);
		}
	}
    // 设置变量，比如 _context,_x,_taps,_interceptors,生成拦截器的call属性
	header() {
		let code = "";
		if (this.needContext()) {
			code += "var _context = {};\n";
		} else {
			code += "var _context;\n";
		}
		code += "var _x = this._x;\n";
		if (this.options.interceptors.length > 0) {
			code += "var _taps = this.taps;\n";
			code += "var _interceptors = this.interceptors;\n";
		}
		for (let i = 0; i < this.options.interceptors.length; i++) {
			const interceptor = this.options.interceptors[i];
			if (interceptor.call) {
				code += `${this.getInterceptor(i)}.call(${this.args({
					before: interceptor.context ? "_context" : undefined
				})});\n`;
			}
		}
		return code;
	}

	needContext() {
		for (const tap of this.options.taps) if (tap.context) return true;
		return false;
	}
    // 具体生成每一个注册函数的字符串
	callTap(tapIndex, { onError, onResult, onDone, rethrowIfPossible }) {
		let code = "";
		let hasTapCached = false;
		for (let i = 0; i < this.options.interceptors.length; i++) {
			const interceptor = this.options.interceptors[i];
			if (interceptor.tap) {
				if (!hasTapCached) {
					code += `var _tap${tapIndex} = ${this.getTap(tapIndex)};\n`;
					hasTapCached = true;
				}
				code += `${this.getInterceptor(i)}.tap(${
					interceptor.context ? "_context, " : ""
				}_tap${tapIndex});\n`;
			}
		}
		code += `var _fn${tapIndex} = ${this.getTapFn(tapIndex)};\n`;
		const tap = this.options.taps[tapIndex];
		switch (tap.type) {
			case "sync":
				if (!rethrowIfPossible) {
					code += `var _hasError${tapIndex} = false;\n`;
					code += "try {\n";
				}
				if (onResult) {
					code += `var _result${tapIndex} = _fn${tapIndex}(${this.args({
						before: tap.context ? "_context" : undefined
					})});\n`;
				} else {
					code += `_fn${tapIndex}(${this.args({
						before: tap.context ? "_context" : undefined
					})});\n`;
				}
				if (!rethrowIfPossible) {
					code += "} catch(_err) {\n";
					code += `_hasError${tapIndex} = true;\n`;
					code += onError("_err");
					code += "}\n";
					code += `if(!_hasError${tapIndex}) {\n`;
        }
                // 如果需要返回值则生成返回值代码
				if (onResult) {
					code += onResult(`_result${tapIndex}`);
        }
                // 新生成的字符串和之前生成的字符串
				if (onDone) {
					code += onDone();
				}
				if (!rethrowIfPossible) {
					code += "}\n";
				}
				break;
			case "async":
				let cbCode = "";
				if (onResult) cbCode += `(_err${tapIndex}, _result${tapIndex}) => {\n`;
				else cbCode += `_err${tapIndex} => {\n`;
				cbCode += `if(_err${tapIndex}) {\n`;
				cbCode += onError(`_err${tapIndex}`);
				cbCode += "} else {\n";
				if (onResult) {
					cbCode += onResult(`_result${tapIndex}`);
				}
				if (onDone) {
					cbCode += onDone();
				}
				cbCode += "}\n";
				cbCode += "}";
				code += `_fn${tapIndex}(${this.args({
					before: tap.context ? "_context" : undefined,
					after: cbCode
				})});\n`;
				break;
			case "promise":
				code += `var _hasResult${tapIndex} = false;\n`;
				code += `var _promise${tapIndex} = _fn${tapIndex}(${this.args({
					before: tap.context ? "_context" : undefined
				})});\n`;
				code += `if (!_promise${tapIndex} || !_promise${tapIndex}.then)\n`;
				code += `  throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise${tapIndex} + ')');\n`;
				code += `_promise${tapIndex}.then(_result${tapIndex} => {\n`;
				code += `_hasResult${tapIndex} = true;\n`;
				if (onResult) {
					code += onResult(`_result${tapIndex}`);
				}
				if (onDone) {
					code += onDone();
				}
				code += `}, _err${tapIndex} => {\n`;
				code += `if(_hasResult${tapIndex}) throw _err${tapIndex};\n`;
				code += onError(`_err${tapIndex}`);
				code += "});\n";
				break;
		}
		return code;
	}

	callTapsSeries({
		onError,
		onResult,
		resultReturns,
		onDone,
		doneReturns,
		rethrowIfPossible
	}) {
		if (this.options.taps.length === 0) return onDone();
		const firstAsync = this.options.taps.findIndex(t => t.type !== "sync");
		const somethingReturns = resultReturns || doneReturns || false;
		let code = "";
		let current = onDone;
		for (let j = this.options.taps.length - 1; j >= 0; j--) {
			const i = j;
			const unroll = current !== onDone && this.options.taps[i].type !== "sync";
			if (unroll) {
				code += `function _next${i}() {\n`;
				code += current();
				code += `}\n`;
				current = () => `${somethingReturns ? "return " : ""}_next${i}();\n`;
			}
			const done = current;
			const doneBreak = skipDone => {
				if (skipDone) return "";
				return onDone();
			};
			const content = this.callTap(i, {
				onError: error => onError(i, error, done, doneBreak),
				onResult:
					onResult &&
					(result => {
						return onResult(i, result, done, doneBreak);
					}),
				onDone: !onResult && done,
				rethrowIfPossible:
					rethrowIfPossible && (firstAsync < 0 || i < firstAsync)
			});
			current = () => content;
		}
		code += current();
		return code;
	}

	callTapsLooping({ onError, onDone, rethrowIfPossible }) {
		if (this.options.taps.length === 0) return onDone();
		const syncOnly = this.options.taps.every(t => t.type === "sync");
		let code = "";
		if (!syncOnly) {
			code += "var _looper = () => {\n";
			code += "var _loopAsync = false;\n";
		}
		code += "var _loop;\n";
		code += "do {\n";
		code += "_loop = false;\n";
		for (let i = 0; i < this.options.interceptors.length; i++) {
			const interceptor = this.options.interceptors[i];
			if (interceptor.loop) {
				code += `${this.getInterceptor(i)}.loop(${this.args({
					before: interceptor.context ? "_context" : undefined
				})});\n`;
			}
		}
		code += this.callTapsSeries({
			onError,
			onResult: (i, result, next, doneBreak) => {
				let code = "";
				code += `if(${result} !== undefined) {\n`;
				code += "_loop = true;\n";
				if (!syncOnly) code += "if(_loopAsync) _looper();\n";
				code += doneBreak(true);
				code += `} else {\n`;
				code += next();
				code += `}\n`;
				return code;
			},
			onDone:
				onDone &&
				(() => {
					let code = "";
					code += "if(!_loop) {\n";
					code += onDone();
					code += "}\n";
					return code;
				}),
			rethrowIfPossible: rethrowIfPossible && syncOnly
		});
		code += "} while(_loop);\n";
		if (!syncOnly) {
			code += "_loopAsync = true;\n";
			code += "};\n";
			code += "_looper();\n";
		}
		return code;
	}

	callTapsParallel({
		onError,
		onResult,
		onDone,
		rethrowIfPossible,
		onTap = (i, run) => run()
	}) {
		if (this.options.taps.length <= 1) {
			return this.callTapsSeries({
				onError,
				onResult,
				onDone,
				rethrowIfPossible
			});
		}
		let code = "";
		code += "do {\n";
		code += `var _counter = ${this.options.taps.length};\n`;
		if (onDone) {
			code += "var _done = () => {\n";
			code += onDone();
			code += "};\n";
		}
		for (let i = 0; i < this.options.taps.length; i++) {
			const done = () => {
				if (onDone) return "if(--_counter === 0) _done();\n";
				else return "--_counter;";
			};
			const doneBreak = skipDone => {
				if (skipDone || !onDone) return "_counter = 0;\n";
				else return "_counter = 0;\n_done();\n";
			};
			code += "if(_counter <= 0) break;\n";
			code += onTap(
				i,
				() =>
					this.callTap(i, {
						onError: error => {
							let code = "";
							code += "if(_counter > 0) {\n";
							code += onError(i, error, done, doneBreak);
							code += "}\n";
							return code;
						},
						onResult:
							onResult &&
							(result => {
								let code = "";
								code += "if(_counter > 0) {\n";
								code += onResult(i, result, done, doneBreak);
								code += "}\n";
								return code;
							}),
						onDone:
							!onResult &&
							(() => {
								return done();
							}),
						rethrowIfPossible
					}),
				done,
				doneBreak
			);
		}
		code += "} while(false);\n";
		return code;
	}

	args({ before, after } = {}) {
		let allArgs = this._args;
		if (before) allArgs = [before].concat(allArgs);
		if (after) allArgs = allArgs.concat(after);
		if (allArgs.length === 0) {
			return "";
		} else {
			return allArgs.join(", ");
		}
	}

	getTapFn(idx) {
		return `_x[${idx}]`;
	}

	getTap(idx) {
		return `_taps[${idx}]`;
	}

	getInterceptor(idx) {
		return `_interceptors[${idx}]`;
	}
}

module.exports = HookCodeFactory;

```

4. 测试生成实例  

+ syncHook  
```js
    const SyncHook = require("../lib/SyncHook");

    // 实例化 SyncHook
    const sh = new SyncHook(['arg1'])

    // 通过 tap 注册 handler
    sh.tap('1', function(arg1, arg2) {
        console.log(arg1, arg2, 1);
    });
    sh.tap({
        name: '2',
        before: '1',
    }, function(arg1) {
        console.log(arg1, 2);
    });
    sh.tap({
        name: '3',
        stage: -1,
    }, function(arg1) {
        console.log(arg1, 3);
    });

    // 通过 call 执行 handler
    sh.call('tapable', 'tapable-2.0.0')

    /**
    * 生成的call函数
    function anonymous(arg1) {
        "use strict";
        var _context;
        var _x = this._x;
        var _fn0 = _x[0];
        _fn0(arg1);
        var _fn1 = _x[1];
        _fn1(arg1);
        var _fn2 = _x[2];
        _fn2(arg1);
    }

    tapable 3
    tapable 2
    tapable undefined 1
    */
```

+ asyncParallelHook_1  
```js
const AsyncParallelHook = require("../lib/AsyncParallelHook");

// 创建实列
const asyncParallelHook = new AsyncParallelHook(["name", "age"]);

// 注册事件
asyncParallelHook.tapAsync("1", (name, age, done) => {
    setTimeout(() => {
        console.log("1", name, age, new Date());
        // done();
        done(1); // 注意done函数，有无函数，以及有无参数 都会影响输出
    }, 1000);
});

asyncParallelHook.tapAsync("2", (name, age, done) => {
    // setTimeout(() => {
    //   console.log("2", name, age, new Date());
    //   done();
    // }, 2000);
    console.log("2", name, age, new Date());
    done(2);
});

asyncParallelHook.tapAsync("3", (name, age, done) => {
    setTimeout(() => {
        console.log("3", name, age, new Date());
        done();
    }, 3000);
});

// 触发事件，让监听函数执行
asyncParallelHook.callAsync("kongzhiEvent-1", 18, () => {
    console.log('函数执行完毕');
});

/**
 * 
function anonymous(name, age, _callback) {
    "use strict";
    var _context;
    var _x = this._x;
    do {
        var _counter = 3;
        var _done = () => {
            _callback();
        };
        if (_counter <= 0) break;
        var _fn0 = _x[0];
        _fn0(name, age, _err0 => {
            if (_err0) {
                if (_counter > 0) {
                    _callback(_err0);
                    _counter = 0;
                }
            } else {
                if (--_counter === 0) _done();
            }
        });
        if (_counter <= 0) break;
        var _fn1 = _x[1];
        _fn1(name, age, _err1 => {
            if (_err1) {
                if (_counter > 0) {
                    _callback(_err1);
                    _counter = 0;
                }
            } else {
                if (--_counter === 0) _done();
            }
        });
        if (_counter <= 0) break;
        var _fn2 = _x[2];
        _fn2(name, age, _err2 => {
            if (_err2) {
                if (_counter > 0) {
                    _callback(_err2);
                    _counter = 0;
                }
            } else {
                if (--_counter === 0) _done();
            }
        });
    } while (false);

}
  
 */


```

+ asyncParallelHook_2  
```js
const AsyncParallelHook = require("../lib/AsyncParallelHook");

// 创建实列
const asyncParallelHook = new AsyncParallelHook(["name", "age"]);

// 注册事件
// 使用tapPromise注册的事件，必须返回一个Promise实列
asyncParallelHook.tapPromise("1", (name, age) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            console.log("1", name, age, new Date());
        }, 1000);
    });
});

asyncParallelHook.tapPromise("2", (name, age) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            console.log("2", name, age, new Date());
        }, 2000);
    });
});

asyncParallelHook.tapPromise("3", (name, age) => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            console.log("3", name, age, new Date());
        }, 3000);
    });
});

// 触发事件，让监听函数执行
asyncParallelHook.promise("kongzhiEvent-1", 18);

/**
 * name,age 
function anonymous(name, age) {
    "use strict";
    return new Promise((_resolve, _reject) => {
        var _sync = true;

        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then(() => { throw _err; }));
            else
                _reject(_err);
        };
        var _context;
        var _x = this._x;
        do {
            var _counter = 3;
            var _done = () => {
                _resolve();
            };
            if (_counter <= 0) break;
            var _fn0 = _x[0];
            var _hasResult0 = false;
            var _promise0 = _fn0(name, age);
            if (!_promise0 || !_promise0.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise0 + ')');
            _promise0.then(_result0 => {
                _hasResult0 = true;
                if (--_counter === 0) _done();
            }, _err0 => {
                if (_hasResult0) throw _err0;
                if (_counter > 0) {
                    _error(_err0);
                    _counter = 0;
                }
            });
            if (_counter <= 0) break;
            var _fn1 = _x[1];
            var _hasResult1 = false;
            var _promise1 = _fn1(name, age);
            if (!_promise1 || !_promise1.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise1 + ')');
            _promise1.then(_result1 => {
                _hasResult1 = true;
                if (--_counter === 0) _done();
            }, _err1 => {
                if (_hasResult1) throw _err1;
                if (_counter > 0) {
                    _error(_err1);
                    _counter = 0;
                }
            });
            if (_counter <= 0) break;
            var _fn2 = _x[2];
            var _hasResult2 = false;
            var _promise2 = _fn2(name, age);
            if (!_promise2 || !_promise2.then)
                throw new Error('Tap function (tapPromise) did not return promise (returned ' + _promise2 + ')');
            _promise2.then(_result2 => {
                _hasResult2 = true;
                if (--_counter === 0) _done();
            }, _err2 => {
                if (_hasResult2) throw _err2;
                if (_counter > 0) {
                    _error(_err2);
                    _counter = 0;
                }
            });
        } while (false);
        _sync = false;
    });
}

 */


```

+ asyncParallelHook_3  
```js
const AsyncParallelHook = require("../lib/AsyncParallelHook");

// 创建实列
const asyncParallelHook = new AsyncParallelHook(["name", "age"]);

// 注册事件
asyncParallelHook.tapAsync("1", (name, age, done) => {
    setTimeout(() => {
        console.log("1", name, age, new Date());
        // done();
        done(1); // 注意done函数，有无函数，以及有无参数 都会影响输出
    }, 1000);
});

asyncParallelHook.tapAsync("2", (name, age, done) => {
    // setTimeout(() => {
    //   console.log("2", name, age, new Date());
    //   done();
    // }, 2000);
    console.log("2", name, age, new Date());
    done(2);
});

asyncParallelHook.tapAsync("3", (name, age, done) => {
    setTimeout(() => {
        console.log("3", name, age, new Date());
        done();
    }, 3000);
});

// 触发事件，让监听函数执行
asyncParallelHook.promise("kongzhiEvent-1", 18).then(() => {
    console.log('函数执行完毕success');
}, err => {
    console.log('函数执行完毕fail', err);
});

/**
 * 
function anonymous(name, age) {
    "use strict";
    return new Promise((_resolve, _reject) => {
        var _sync = true; // 顺序执行标识~
        function _error(_err) {
            if (_sync)
                _resolve(Promise.resolve().then(() => { throw _err; }));
            else
                _reject(_err);
        };
        var _context;
        var _x = this._x;
        do {
            var _counter = 3;
            var _done = () => {
                _resolve();
            };
            if (_counter <= 0) break;
            var _fn0 = _x[0];
            _fn0(name, age, _err0 => {
                if (_err0) {
                    if (_counter > 0) {
                        _error(_err0);
                        _counter = 0;
                    }
                } else {
                    if (--_counter === 0) _done();
                }
            });
            if (_counter <= 0) break;
            var _fn1 = _x[1];
            _fn1(name, age, _err1 => {
                if (_err1) {
                    if (_counter > 0) {
                        _error(_err1);
                        _counter = 0;
                    }
                } else {
                    if (--_counter === 0) _done();
                }
            });
            if (_counter <= 0) break;
            var _fn2 = _x[2];
            _fn2(name, age, _err2 => {
                if (_err2) {
                    if (_counter > 0) {
                        _error(_err2);
                        _counter = 0;
                    }
                } else {
                    if (--_counter === 0) _done();
                }
            });
        } while (false);
        _sync = false;
    });
}

  // --- console-----
  2 kongzhiEvent-1 18
  Sat Feb 01 2020 00:31:45 GMT+0800 (CST) {}

  函数执行完毕fail 2

  1 kongzhiEvent-1 18
  Sat Feb 01 2020 00:31:46 GMT+0800 (CST) {}
 */


```

+ 利用__tests__里debug测试文件  
```json
    // launch.json
    {
      "type": "node",
      "request": "launch",
      "name": "tapable的jest调试",
      "cwd": "${workspaceFolder}/tapable",
      "skipFiles": [
        "<node_internals>/**"
      ],
      "program": "${workspaceFolder}/tapable/lib/__tests__/AsyncParallelHooks.js",
      "outFiles": [
        "${workspaceFolder}/**/*.js"
      ],
      "runtimeExecutable": "npm",// node 的可执行程序
      "runtimeArgs": [ // 类似于命令行参数
          "run-script", "debug"
      ],
      "port": 9229,
      // 使用什么终端工具。integratedTerminal 表示使用 Visual Studio Code 内置的终端工具，也可以配置成使用操作系统默认的终端工具。
      "console": "integratedTerminal"
    }
    // package.json
    "scripts": {
        "debug": "node --inspect-brk ./node_modules/jest/bin/jest --runInBand --no-cache --no-watchman"
    },
```

5. 低版本兼容  
版本2xx以上已经没有Tapable.js文件，但由于webpack4用的是1xx，其中Compiler extends Tapable，而且插件注册也是提供带有 apply 方法的对象，里面利用compiler.plugin来注册，所以需要兼容  

```js
/* 创建插件 */
// 一个 JavaScript 命名函数。
function MyExampleWebpackPlugin() {

};
// 在插件函数的 prototype 上定义一个 `apply` 方法。
MyExampleWebpackPlugin.prototype.apply = function(compiler) {
  // 指定一个挂载到 webpack 自身的事件钩子。（其实这里，实际上执行的是Tapable的tap注册）
  compiler.plugin('webpacksEventHook', function(compilation /* 处理 webpack 内部实例的特定数据。*/, callback) {
    console.log("This is an example plugin!!!");

    // 功能完成后调用 webpack 提供的回调。
    callback();
  });
};
```

```js
    // Tapable.js
    const util = require("util");
    const SyncBailHook = require("./SyncBailHook");

    function Tapable() {
        // 声明同步保险钩子
        this._pluginCompat = new SyncBailHook(["options"]);
        // 注册 handler，主要是为了将 pluginName camelize 化。
        this._pluginCompat.tap(
            {
                name: "Tapable camelCase",
                stage: 100
            },
            options => {
                options.names.add(
                    options.name.replace(/[- ]([a-z])/g, (str, ch) => ch.toUpperCase())
                );
            }
        );
        // 在 hooks 属性上对应的钩子上注册 handler
        this._pluginCompat.tap(
            {
                name: "Tapable this.hooks",
                stage: 200
            },
            options => {
                let hook;
                for (const name of options.names) {
                    hook = this.hooks[name];
                    if (hook !== undefined) {
                        break;
                    }
                }
                if (hook !== undefined) {
                    const tapOpt = {
                        name: options.fn.name || "unnamed compat plugin",
                        stage: options.stage || 0
                    };
                    if (options.async) hook.tapAsync(tapOpt, options.fn);
                    else hook.tap(tapOpt, options.fn);
                    return true;
                }
            }
        );
    }
    module.exports = Tapable;

    Tapable.addCompatLayer = function addCompatLayer(instance) {
        Tapable.call(instance);
        instance.plugin = Tapable.prototype.plugin;
        instance.apply = Tapable.prototype.apply;
    };

    // 注册 handler，实际上会走到 _pluginCompat 属性上的第二个 handler，进而在对应的 hooks 注册了 handler。
    Tapable.prototype.plugin = util.deprecate(function plugin(name, fn) {
        if (Array.isArray(name)) {
            name.forEach(function(name) {
                this.plugin(name, fn);
            }, this);
            return;
        }
        const result = this._pluginCompat.call({
            name: name,
            fn: fn,
            names: new Set([name])
        });
        if (!result) {
            throw new Error(
                `Plugin could not be registered at '${name}'. Hook was not found.\n` +
                    "BREAKING CHANGE: There need to exist a hook at 'this.hooks'. " +
                    "To create a compatibility layer for this hook, hook into 'this._pluginCompat'."
            );
        }
    }, "Tapable.plugin is deprecated. Use new API on `.hooks` instead");

    Tapable.prototype.apply = util.deprecate(function apply() {
        for (var i = 0; i < arguments.length; i++) {
            arguments[i].apply(this);
        }
    }, "Tapable.apply is deprecated. Call apply on the plugin directly instead");

```