title: 在JavaScript中实现AOP
date: 2016-01-17 16:57:37
tags: [JavaScript, AOP]
---
有一个公用的代码，可能在很多地方都会被用到，那么现在要做的就是，需要这个方法跑起来之前走一些东西，在这个方法跑完之后，还在处理一些东西。
<!-- more -->
## 概念解读
AOP(Aspect Oriented Programming)，面向切面编程，主要实现的目的是针对业务处理过程中的切面进行提取，它所面对的是处理过程中的某个步骤或阶段，以获得逻辑过程中各部分之间低耦合性的隔离效果。
AOP的主要思想是把一些与核心业务无关但又在多个模块使用的功能分离出来，然后动态给业务模块添加上 需要的功能。
AOP可以实现对业务无侵入式地干扰。

## 代码实战
如果我们想得到某个函数的执行的开始时间和结束时间，可能会写出如下的代码：
```
    function test1(){
        console.log(1);
    }
    console.log('test1 start:', Date.now());
    test1();
    console.log('test2 start:', Date.now());
```
上面的代码确实能完成响应的需求，但是扩展性不好，倘若遇到需要得到多个函数它们执行的开始和结束时间，如果还是按上面的那种方式，几乎同样的代码会写很多遍，如下：
```
    function test1(){
        console.log(1);
    }
    function test2(){
        console.log(2);
    }
    function test3(){
        console.log(3);
    }
    ...
    console.log('test1 start:', Date.now());
    test1();
    console.log('test1 start:', Date.now());

    console.log('test2 start:', Date.now());
    test2();
    console.log('test2 start:', Date.now());

    console.log('test3 start:', Date.now());
    test3();
    console.log('test3 start:', Date.now());
    ...
```
像这样很不优雅，代码变得极难维护，这时候若实现AOP就能够很轻松的解决类似的问题。
```
    function test1(){
        console.log(1);
    }

    Function.prototype.before = function(fn){
        var self = this;  
        fn(self.name); 
        self.apply(this, arguments);         
    }
    Function.prototype.after= function(fn){
        var self = this;
        self.apply(this, arguments);   
        fn(self.name);
    }
    test1.before(printTime);
    test1.after(printTime);

    function printTime () {
        console.log(arguments[0], 'end:', Date.now());
    }    
```
每个函数都是Function类型的实例，因为Function通过prototype挂载了before方法和after方法，所以作为Function的实例，test1函数就拥有了before和after方法。现在执行test1.before(printTime)和test1.after(printTime)就会自动打印函数执行的开始和结束时间。
但是，在上面的代码中发现test1执行了两次，可以用test1函数作为中转，通过执行before后将before的回调和before一起送到after去来解决这个问题。
```
    var funcName;
    function test1(){
        console.log(1);
    }
    Function.prototype.before = function(fn){
        var self = this;
        funcName = self.name;
        return function(){
            fn.apply(this, arguments);  
            self.apply(self, arguments);
        };
    };

    Function.prototype.after = function(fn){
        var self = this;
        return function(){
            self.apply(self, arguments);
            fn.apply(this, arguments);
        };
    };

    test1.before(printTime).after(printTime)();  

    function printTime () {
        console.log(funcName, 'end:', Date.now());
    } 
```
在上面的代码中，test1函数调用before方法后得到一个匿名函数，该匿名函数继续调用其拥有的after方法，又得到一个匿名函数，最后执行这个匿名函数。最后这个匿名函数做的工作有打印test1()开始执行前的时间，执行test1函数，打印test1()执行后的时间。
这样就通过before中的回调作为传递，test1就只执行一次。
像最开始的需求那样，若有多个函数需要做相同的事，只需像下面这样就能做到。
```
    test1.before(printTime).after(printTime)();
    test2.before(printTime).after(printTime)();
    test3.before(printTime).after(printTime)();
    .
    .
    .
    testN.before(printTime).after(printTime)();
```
