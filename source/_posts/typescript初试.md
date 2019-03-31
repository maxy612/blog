---
title: TypeScript是什么？
date: 2017-10-16 18:17:19
tags: ["TypeScript"]
summary: "本文主要讲述TypeScript的由来及简单用法，也解决了之前看vue源码时的一些疑惑，更详细地教程请看本文最后的参考链接。"
category: "学习"
---

+ 问题由来
+ TypeScript概述及简单用法
+ TypeScript学习资源

## 问题由来
最近Vue更新到2.5版本了，看到Vue加强了对TypeScript的集成优化。虽然对优化的细节还不清楚，但是在想TypeScript是个什么东西？之前暑期实习的时候，曾经翻过Vue的部分源码，源码里就有用到了TypeScript，那时候还疑惑js怎么还可以这样写。基于此，我觉得有必要尝试一下TypeScript。

## TypeScript的概述和简单用法
TypeScript是JavaScript的超集（也就是JavaScript是TypeScript的子集）。TypeScript的后缀是.ts或者.tsx，通过全局安装typescript将ts或tsx文件编译成.js文件。
既然有了JavaScript，为什么还需要TypeScript呢？
JavaScript是一门弱类型语言，用法相较于c, c++, Java等这些语言比较灵活。如果一开始就接触的是JavaScript,会习惯于它的灵活的用法。另一方面，JavaScript也是一门解释型语言，这也就意味着js脚本中的错误只有在执行的时候才会在浏览器中出现，对于由c, c++, Java转到JavaScript的开发人员，无疑很不方便。而TypeScript可以在写代码和编译为js的时候对代码进行检查，能够提前发现代码中的错误，这样极大方便了开发；同时，在TypeScript里，可以使用类似ES6的语法，TypeScript可以编译成es3或者es5的代码。

那下面就来简单地学习一些TypeScript的入门知识吧。
首先，全局安装TypeScript（首先要保证自己电脑上安装了nodejs和npm）。
{% codeblock lang:shell %}
// 安装typescript
npm install -g typescript

// 新建项目文件夹并进入
mkdir typescript-learn && cd typescript-learn
{% endcodeblock %}

#### 函数
平时我们写js函数，一般是这样的：
{% codeblock lang:javascript %}
function Point(x, y) {
  if (typeof x === 'number' && typeof y === 'number' && !isNaN(x) && !isNaN(y)) {
    throw Error('x, y必须为数字类型');
  }
  return Math.sqrt(x * x + y * y);
}
{% endcodeblock %}

通常情况下，我们需要通过if判断来限制参数类型；如果我们使用TypeScript，可以这样：
{% codeblock lang:javascript %}
function Point(x: number, y: number): number {
  return Math.sqrt(x ** 2 + y ** 2);
}
{% endcodeblock %}

在typescript-learn文件夹中，创建一个test.ts，写入上面的函数。当你调用该函数时，

{% codeblock lang:javascript %}
// 故意放入一个错误类型的字符串
Point('333', 3); // error TS2345: Argument of type '"333"' is not assignable to parameter of type 'number'
Point(3, 2); // ok
{% endcodeblock %},

这时我们看下编译后生成的js文件

{% codeblock lang:javascript %}
function Point(x, y) {
    return Math.sqrt(Math.pow(x, 2) + Math.pow(y, 2));
}
Point(33, 3);
{% endcodeblock %}

经过TypeScript编译后，生成了我们熟悉的js文件和语法。在这个过程中，你会发现，如果代码中有错误，在命令行就会显示出来方便我们处理；此外，代码被转换成了es5的语法，兼容性也一并解决了。

#### 变量

函数声明可以采用上边的方式，变量声明同样也可以。

{% codeblock lang:javascript %}
let a: number = 30; // ok
let b: boolean = true; // ok
let c: number[] = [3, 4, 5]; // ok
let d: string = `hello, world ${a}`; // ok
let e: any = 'dddd'; // ok  如果不清楚变量的类型，可以用any，表示任何类型
{% endcodeblock %}

这些代码会被编译为：

{% codeblock lang:javascript %}
var a = 30; // ok
var b = true; // ok
var c = [3, 4, 5]; // ok
var d = "hello, world " + a; // ok
var e = 'dddd'; // ok  如果不清楚变量的类型，可以用any，表示任何类型
{% endcodeblock %}

此外，在TypeScript中，还有一种新的数据类型---枚举
{% codeblock lang:javascript %}
enum Color {
  Green,
  Blue,
  Red
}

console.log(Color);

// 会被编译成这样：
var Color;
(function (Color) {
    Color[Color["Green"] = 0] = "Green";
    Color[Color["Blue"] = 1] = "Blue";
    Color[Color["Red"] = 2] = "Red";
})(Color || (Color = {}));

console.log(Color); // { '0': 'Green', '1': 'Blue', '2': 'Red', Green: 0, Blue: 1, Red: 2 }
{% endcodeblock %}

通过这样的方式，会生成一个对象，在键名和键值之间建立双向映射。当然，如果给Green, Blue赋值时，如果赋值为其它数字，会在赋值的数字和Green, Blue建立双向映射。

{% codeblock lang:javascript %}
enum Color {
    Green = 4,
    Blue = 6,
    Red = 'hello'
}

// 编译过后，生成
var Color;
(function (Color) {
    Color[Color["Green"] = 4] = "Green";
    Color[Color["Blue"] = 6] = "Blue";
    Color["Red"] = "hello";
})(Color || (Color = {}));
{% endcodeblock %}

#### 类

熟悉JavaScript的人都知道，JavaScript的类不是真正的类，而是伪类，是通过构造函数来实现的。尽管在ES6中实现了class的语法糖，但仍然不是真正的类。在Java或者C++中，我们编写类，会涉及到变量的不同类型，如 public, protected, private等，数据的作用范围和类型有严格的限制。在javascript中，我们写构造函数时，并不能通过public, protected, private来对变量或函数进行控制。但是在typescript中，我们却可以这样：

{% codeblock lang:javascript %}
class Person {
    private name: string; // 只能在该类内部可用
    public hobby: string[]; // 可自由调用
    protected job: string; // 在类内部和子类中可用

    constructor(name: string, hobby: string | string[]) {
        this.name = name;

        if (typeof hobby === 'string') {
            this.hobby = [hobby];
        } else if (Array.isArray(hobby)) {
            this.hobby = hobby;
        }
    }

    getHobby() {
        return this.hobby;
    }

    getName() {
        return this.name;
    }

    editName(newName: string) {
        this.name = newName;
    }
}

class Doctor extends Person {
    constructor(name: string, hobby: string | string[], job?: string /* 后边加问号代表参数可选 */) {
        super(name, hobby);
        this.job = job || '自由职业者';
    }

    getJob() {
        return this.job;
    }
}

var doctor = new Doctor('maxy', 'coding', 'doctor');
console.log(doctor); // Doctor { name: 'maxy', hobby: [ 'coding' ], job: 'doctor' }
console.log(doctor.name); // error TS2341: Property 'name' is private and only accessible within class 'Person'.
console.log(doctor.hobby); // [ 'coding' ]
{% endcodeblock %}

相信有过面向对象编程经验的人能够看懂用typescript实现的代码.

#### 接口

TypeScript的核心原则之一是对值所具有的结构进行类型检查。而接口可以定义数据的结构，在使用所定义的数据结构时typescript会进行检查。

{% codeblock lang:javascript %}
interface Person {
    readonly name: string;
    sex: string;
    hobby: string | string[];
    job?: string;
}

let person: Person = {
    name: 'maxy',
    sex: 'male',
    hobby: 'codiing',
    job: '333'
}

function fun(person: Person) {
  this.name = person.name;
  this.sex = person.sex;
  this.hobby = person.hobby;
}

fun({
    name: 'dddd'
}) // Argument of type '{ name: string; }' is not assignable to parameter of type 'Person' Property 'sex' is missing in type '{ name: string; }'.

person.name = 'zhang'; // Error: Cannot assign to 'name' because it is a constant or a read-only property
{% endcodeblock %}

通过接口这种方式，可以很方便的定义和调用结构化的数据。

除了以上这些，还有命名空间，遍历器，模块化，类型推断等更多功能，但上边这些应该是经常用到的，感兴趣的话可以再下面的学习链接中继续探索。

## TypeScript学习资源

参考链接：
+ {% link typescript-handbook    https://zhongsp.gitbooks.io/typescript-handbook/content/doc/handbook/tutorials/index.html
 [external] [typescript handbook] %}
+ {% link typescript官网 https://www.typescriptlang.org/ [external] [typescript官网] %}
