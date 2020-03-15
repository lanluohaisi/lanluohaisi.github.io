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

// JavaScript 没有空值（Void）的概念，在 TypeScript 中，可以用 void 表示没有任何返回值的函数
function logName(): void {
  console.log('My name is Tom');
}

// 如果定义的时候没有赋值，不管之后有没有赋值，都会被推断成 any 类型而完全不被类型检查：
let myFavoriteNumber;
myFavoriteNumber = 'seven';
myFavoriteNumber = 7;

//访问 string 和 number 的共有属性是没问题的：
function getString(something: string | number): string {
    return something.toString();
}

// 接口
interface Person {
  name: string;
  age?: number;
  // 一旦定义了任意属性，那么确定属性和可选属性的类型都必须是它的类型的子集
  [propName: string]: string | number | undefined;
}

let tom: Person = {
  name: 'Tom',
  age: 25,
  gender: 'male'
};

// 数组
let fibonacci: number[] = [1, 1, 2, 3, 5]; // 最简单的方法是使用「类型 + 方括号」来表示数组
let fibonacci1: Array<number> = [1, 1, 2, 3, 5]; // 数组泛型
function sum() {
  // 事实上常用的类数组都有自己的接口定义，如 IArguments, NodeList, HTMLCollection 等：
  let args: IArguments = arguments;
}

// 函数
function sum1(x: number, y: number): number {
  return x + y;
}
let mySum: (x: number, y: number) => number = function (x: number, y: number): number {
  return x + y;
};
// 函数重载
function reverse(x: number): number;
function reverse(x: string): string;
function reverse(x: number | string): number | string | undefined {
    if (typeof x === 'number') {
        return Number(x.toString().split('').reverse().join(''));
    } else if (typeof x === 'string') {
        return x.split('').reverse().join('');
    }
}

// 类型断言 <类型>值 或者 值 as 类型
function getLength(something: string | number): number {
  if ((\<string\>something).length) { // \号转义，防止md文件失效
      return (\<string\>something).length;
  } else {
      return something.toString().length;
  }
}

// 声明文件
// 需要注意的是，声明语句中只能定义类型，切勿在声明语句中定义具体的实现
// 除了全局变量之外，可能有一些类型我们也希望能暴露出来。在类型声明文件中，我们可以直接使用 interface 或 type 来声明一个全局的接口或类型
/**
 *  declare var 声明全局变量
    declare function 声明全局方法
    declare class 声明全局类
    declare enum 声明全局枚举类型
    declare namespace 声明（含有子属性的）全局对象
    interface 和 type 声明全局类型
 */
declare const jQuery: (selector: string) => any;
declare function jQuery1(selector: string): any;
declare class Animal1 {
  name: string;
  constructor(name: string);
  sayHi(): string;
}
interface AjaxSettings {
  method?: 'GET' | 'POST'
  data?: any;
}
interface Alarm {
  alert() : undefined;
}

// 类型别名
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

// 字符串字面量类型
type EventNames = 'click' | 'scroll' | 'mousemove';
function handleEvent(ele: HTMLElement | null, event: EventNames) {
    // do something
}
handleEvent(document.getElementById('hello'), 'scroll');  // 没问题

// 元组
// 数组合并了相同类型的对象，而元组（Tuple）合并了不同类型的对象
let tom1: [string, number] = ['Tom', 25];

// 枚举
enum Days {Sun, Mon, Tue, Wed, Thu, Fri, Sat};
const enum Directions {
  Up,
  Down = 19,
  Left,
  Right
}
let directions = [Directions.Up, Directions.Down, Directions.Left, Directions.Right];

// 类
class Animal {
  public name : string;
  public constructor(name : string) {
      this.name = name;
  }
  sayHi() {
    return `My name is ${this.name}`;
  }
}
class Cat extends Animal {
  constructor(name : string) {
    super(name);
  }
  sayHi() {
    return 'Meow, ' + super.sayHi(); // 调用父类的 sayHi()
  }
}
let a = new Animal('Jack');
console.log(a.name);
a.name = 'Tom';
let c = new Cat('Tom'); // Tom
console.log(c.sayHi()); // Meow, My name is Tom

// 泛型--泛型（Generics）是指在定义函数、接口或类的时候，不预先指定具体的类型，而在使用的时候再指定类型的一种特性
function swap<T, U>(tuple: [T, U]): [U, T] {
  return [tuple[1], tuple[0]];
}
swap([7, 'seven']); // ['seven', 7]
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


关于ts类继承实现                {#ts_extends}
------------------------------------

**注意attention:** [学习参考文档：TypeScript高级用法的知识点汇总](https://www.jb51.net/article/176528.htm)

```ts
// 源ts文件：
class Animal {
    public name : string;
    public constructor(name : string) {
        this.name = name;
    }
    sayHi() {
      return `My name is ${this.name}`;
    }
}
class Cat extends Animal {
    constructor(name : string) {
      super(name);
    }
    sayHi() {
      return 'Meow, ' + super.sayHi(); // 调用父类的 sayHi()
    }
}
let a = new Animal('Jack');
console.log(a.name);
a.name = 'Tom';
let c = new Cat('Tom'); // Tom
console.log(c.sayHi()); // Meow, My name is Tom
```

```js
// 转换后的js文件
var __extends = (this && this.__extends) || (function () {
    var extendStatics = function (d, b) {
        extendStatics = Object.setPrototypeOf ||
            ({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||
            function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };
        return extendStatics(d, b);
    };
    return function (d, b) {
        extendStatics(d, b);
        function __() { this.constructor = d; }
        d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
    };
})();


var Animal = /** @class */ (function () {
    function Animal(name) {
        this.name = name;
    }
    Animal.prototype.sayHi = function () {
        return "My name is " + this.name;
    };
    return Animal;
}());
var Cat = /** @class */ (function (_super) {
    __extends(Cat, _super);
    function Cat(name) {
        return _super.call(this, name) || this;
    }
    Cat.prototype.sayHi = function () {
        return 'Meow, ' + _super.prototype.sayHi.call(this); // 调用父类的 sayHi()
    };
    return Cat;
}(Animal));
var a = new Animal('Jack');
console.log(a.name);
a.name = 'Tom';
var c = new Cat('Tom'); // Tom
console.log(c.sayHi()); // Meow, My name is Tom
```

解释：
+ 第一部分--`extendStatics(d, b)`，在extendStatics(d, b)方法中，d指子类Child，b指父类Parent，因此该方法的作用可以解释为:  
```js
// 将子类Child的__proto__属性指向父类Parent
Child.__proto__ = Parent;
// 可以将这行代码理解为构造函数的继承，或者叫静态属性和静态方法的继承，即属性和方法不是挂载到构造函数的prototype原型上的，而是直接挂载到构造函数本身,通过这种方式来实现静态属性和静态方法的路径查找
```
+ 在第二部分中仅包含以下两行代码:  
```js
function __() { this.constructor = d; }
d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());

// 其中d指子类Cat，b指父类Animal，这里对于JS中实现继承的几种方式比较熟悉的同学可以一眼看出，这里使用了寄生组合式继承的方式，通过借用一个中间函数__()来避免当修改子类的prototype上的方法时对父类的prototype所造成的影响。我们知道，在JS中通过构造函数实例化一个对象之后，该对象会拥有一个__proto__属性并指向其构造函数的prototype属性

// 可以理解为以下: 
const cat = new Cat();
cat.__proto__ === (Cat.prototype = new __());
cat.__proto__.__proto__ === __.prototype === Animal.prototype;
 
// 上述代码等价于下面这种方式
Cat.prototype.__proto__ === Animal.prototype;
```

+ vue里面的继承  
```js
// 构造VueComponent，Vue子类
    const Sub = function VueComponent (options) {
      this._init(options)
    }
    // 将vue上原型的方法挂在Sub.prototype中，Sub的实例同时也继承了vue.prototype上的所有属性和方法。
    // 关于 prototype的学习：http://www.cnblogs.com/dolphinX/p/3286177.html
    Sub.prototype = Object.create(Super.prototype)
    // Sub构造函数修正，学习于https://www.cnblogs.com/SheilaSun/p/4397918.html
    Sub.prototype.constructor = Sub
```

+ 组合式继承:  
```js
function Person(name){
    this.name=name;
}
Person.prototype.sayName=function(){
    console.log(this.name+' '+this.gender+' '+this.age);
}
function Female(name,gender,age){
    Person.call(this,name);//第二次调用父类构造函数
    this.age=age;
    this.gender=gender;
}
Female.prototype=new Person();//第一次调用父类构造函数
Female.prototype.constrcutor=Female;//因重写原型而失去constructor属性，所以要对constrcutor重新赋值

```

+ 寄生组合式继承  
即通过借用构造函数来继承属性，通过原型链的方式来继承方法，而不需要为子类指定原型而调用父类的构造函数，我们需要拿到的仅仅是父类原型的一个副本。因此可以通过传入子类和父类的构造函数作为参数，首先创建父类原型的一个复本，并为其添加constrcutor，最后赋给子类的原型。这样避免了调用两次父类的构造函数，为其创建多余的属性。
```js
function inheritPrototype(Female,Person){ 
    var protoType=Object.create(Person.prototype);
    protoType.constructor=Female;
    Female.prototype=protoType;
}
//取代
//Female.prototype=new Person();
//Female.prototype.constrcutor=Female

/*在ES5中规范了Object.create()方法*/
function object(o){
    function f(){}
    f.prototype = o;
    return new f();
}
```

