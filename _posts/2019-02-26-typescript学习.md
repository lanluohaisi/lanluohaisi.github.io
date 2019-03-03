---
layout: post
title:  "2019-02-26-typescript学习"
date:   2019-02-26 10:36:01 +0800
categories: js
tag: javascript
---

* content
{:toc}

官网英文  [ http://www.typescriptlang.org/](http://www.typescriptlang.org/)  
中文  [https://www.tslang.cn/docs/home.html](https://www.tslang.cn/docs/home.html)  
学习参考文档  [https://github.com/xcatliu/typescript-tutorial/blob/master/README.md](https://github.com/xcatliu/typescript-tutorial/blob/master/README.md)  

初始                {#ts_init}
------------------------------------

```javascript
function sayHello(person: string) {
    return 'Hello, ' + person;
}

let user: string = 'Tom';
let isDone: boolean = false;
let anyThing: any = 'Tom';

// 如果定义的时候没有赋值，不管之后有没有赋值，都会被推断成 any 类型而完全不被类型检查：
let myFavoriteNumber;
myFavoriteNumber = 'seven';
myFavoriteNumber = 7;

//访问 string 和 number 的共有属性是没问题的：
function getString(something: string | number): string {
    return something.toString();
}

let fibonacci: Array<number> = [1, 1, 2, 3, 5];

console.log(sayHello(user));

// type类型别名: 用来给一个类型起个新名字
type Name = string;
type NameResolver = () => string;
type NameOrResolver = Name | NameResolver;
function getName(n: NameOrResolver): Name {
    if (typeof n === 'string') {
        return n;
    } else {
        return n();
    }
}
// 枚举（Enum）类型用于取值被限定在一定范围内的场景
enum Days {Sun, Mon, Tue, Wed, Thu, Fri, Sat};
```

什么是声明文件                {#ts_tds}
------------------------------------

通常我们会把声明语句放到一个单独的文件（jQuery.d.ts）中，这就是声明文件;声明文件必需以 .d.ts 为后缀。
一般来说，ts 会解析项目中所有的 *.ts 文件，当然也包含以 .d.ts 结尾的文件。所以当我们将 jQuery.d.ts 放到项目中时，其他所有 *.ts 文件就都可以获得 jQuery 的类型定义了。  

一般我们通过 import foo from 'foo' 导入一个 npm 包，这是符合 ES6 模块规范的。  
在我们尝试给一个 npm 包创建声明文件之前，首先看看它的声明文件是否已经存在。一般来说，npm 包的声明文件可能存在于两个地方：  

1. 与该 npm 包绑定在一起。判断依据是 package.json 中有 **types 或者 typings** 字段，或者有一个 index.d.ts 声明文件。这种模式不需要额外安装其他包，是最为推荐的，所以以后我们自己创建 npm 包的时候，最好也将声明文件与 npm 包绑定在一起。  
2. 发布到了 @types 里。只要尝试安装一下对应的包就知道是否存在，安装命令是 npm install @types/foo --save-dev。这种模式一般是由于 npm 包的维护者没有提供声明文件，所以只能由其他人将声明文件发布到 @types 里了。  

假如以上两种方式都没有找到对应的声明文件，那么我们就需要自己为它写声明文件了。由于是通过 import 语句导入的模块，所以声明文件存放的位置也有所约束，一般有两种方案：  

1. 创建一个 node_modules/@types/foo/index.d.ts 文件，存放 foo 模块的声明文件。这种方式不需要额外的配置，但是 node_modules 目录不稳定，代码也没有被保存到仓库中，无法回溯版本，有不小心被删除的风险。
2. 创建一个 types 目录，专门用来管理自己写的声明文件，将 foo 的声明文件放到 types/foo/index.d.ts 中。这种方式需要配置下 tsconfig.json 的 paths 和 baseUrl 字段。  




