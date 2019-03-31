---
title: Node学习————模块机制
date: 2018-01-13 20:41:41
tags: ["node"]
summary: 平时在用express开发网站后台应用时离不开require，长久以来没有对require进行进一步的思考，不太了解它的机制和历史。在这篇文章中，对node的模块机制进行说明。
---

关于node的模块机制，我主要说三个问题：
1. 为什么node会有模块机制？
2. node的模块机制是什么呢？
3. 模块机制为我们带来了什么？

-------------------------------
1. 为什么node会有模块机制？
首先要说明的是，node是javascript，而早期的javascript作为脚本语言，主要用于处理网页的交互，修改dom，它的主要引入方式是通过<script src=""></script>。对于一般的复杂度低的页面交互和处理，已经够用了。但是它不能像其它语言一样开发大型应用。其中一个原因是因为它缺乏对文件有效的管理和组织。如果js文件之间有引用，通常是暴露全局变量，利用加载顺序来调用。对于其它适合开发大型应用的语言（java, python, c/c++等）来说，它们都可以通过自身的模块机制对代码进行抽象封装，实行有效的组织管理。相比之下，javascript缺乏模块系统，一定程度上限制了它的能力。但是对于javascript的探索，社区在不断地尝试javascript在服务端的应用，提出了commonjs规范。node在开发时参考了commonjs的模块系统规范，实现了javascript的模块系统，与npm相配合，大大拓展了javascript的能力。

2. node的模块机制是什么？
首先，模块机制不是nodejs独有的，在java中有类文件，python中有import,php中有include和require,这些都是模块机制。node的模块机制是以commonjs规范为基础的文件组织管理系统，它便利了我们对javascript文件组织和调用，同时为单个javascript提供了独立的作用域。

  在node中调用模块，主要包含三个步骤：路径分析->文件定位->编译执行。我们在使用require引入文件时，需要在require()中传入引用文件的路径，一个文件就是一个模块。模块分为核心模块和文件模块。核心模块是node自带的模块，底层一般是基于c/c++，在node源代码编译时被编译为二进制文件。在调用核心模块时，核心模块文件会直接被加载进内存中且无需编译，速度快。文件模块是用户编写的文件。一般来说，单个文件就可以视为一个模块，可以通过require进行调用，这类模块在调用时需要先通过传入的路径进行文件定位，找到对应的文件后进行同步编译执行，然后返回给调用者，在调用时它是从磁盘调取的，因而速度较核心模块慢。

  无论是核心模块还是文件模块，都有模块缓存机制。在初次加载模块后，node会对它们进行缓存，存在NativeModule._cache和Module._cache中。再次加载时，会自动从缓存中加载模块，能极大的提高模块调用的效率。

  路径分析中，node会对传入require的路径进行解析，一般有require('fs')，require('./a')，require('/b')的形式，第一种一般是加载核心模块或第三方模块，第二三种是加载用户自己的本地模块。第一类的模块查找中，会从当前目录的node_modules开始查，如果没有，依次查找父级目录的node_modules目录直到根目录，当然，核心模块直接从内存中查找。第二类是查找当前目录下的a.js或a.json或a.node文件。第三类是从根目录查找文件。
  定位到对应目录后，进行文件定位。require后的路径中，可以加文件后缀，也可以不加。但是还是加上好，这样node就可以直接查找文件。如果没有加扩展名，node将会依次查找定位目录下的js，json，node文件。比如require('./a')，会依次查找当前目录下a.js，a.json，a.node，由于判断文件存在的过程是通过fs同步加载文件来判断的，因而会影响效率，所以最好是指明后缀。如果没有发现文件，发现了a目录，将会查找a目录的package.json文件，通过JSON.parse找到json文件的main，如果不存在main属性或者package.json文件不存在，会依次查找a目录下的index.js，index.json，index.node文件。

  定位到文件以后，node会建立一个模块对象，然后根据载入路径来编译和执行。对于.js，.json，.node文件，有不同的加载和执行策略。对于js文件，会利用fs.readFileSync进行同步读取和编译，而对于json文件，通过fs同步读取文件后将文件内容通过JSON.parse进行格式化后返回结果。对于node文件，通过process.dlopen来动态加载编译生成的文件。在每个文件的require对象下，有extensions属性，里边有对'.js'，'.json'，'.node'文件的处理函数，可以验证上边所说的不同文件的加载和执行策略。

  在写js文件时，我们可以直接使用require,但是我们之前并没有定义过，这就涉及到模块机制的js解析编译过程。在我们写的js文件内容之上，node会在编译时会将我们写的内容提取出来，然后在内容的前后分别加上
  {% codeblock lang:javascript %}
    (function (exports, require, module, __filename, __dirname) {
       // ...
       exports.length = 30;
    })
  {% endcodeblock %}
  中间引号省略的是内容，然后通过node内置的vm模块的runInThisContext（类似于eval）方法运行处理后的内容，生成一个函数，然后将当前模块对象的exports, require, module, __dirname, __filename传入函数，最后返回exports给调用者。所以我们通过require可以获得所调用模块的exports对象。

3. 模块机制为我们带来了什么？
通过利用node的模块机制，我们能够很方便的组织js文件，为我们利用javascript开发大型应用，客户端提供了可能。仅仅有node的模块系统还不够，我们只能在本地管理利用自己开发的模块。对于他人开发的优秀的第三方模块，我们不能直接使用。npm作为node的包管理工具，使我们可以利用第三方模块完成日常的开发工作，同时我们也可以发布自己的模块到npm上，供他人使用。npm的出现也使得javascript的社区更加强大，资源更加丰富。利用这些，我们能够做更多好玩的东西了。
