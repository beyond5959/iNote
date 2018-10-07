title: 'yield和yield*'
date: 2015-12-18 16:29:52
tags: [JavaScript, Node.js]
---
在ES6的generator函数中经常能见到yield和yield*，开始不是很清楚它们之间有什么区别，在参考了一些资料过后渐渐有一些清楚了。
<!-- more -->
## Array与String
```
function* GenFunc(){
    yield [1, 2];
    yield* [3, 4];
    yield "56";
    yield* "78";
}

var gen = GenFunc();
console.log(gen.next().value);  //[1, 2]
console.log(gen.next().value);  //3
console.log(gen.next().value);  //4
console.log(gen.next().value);  //"56"
console.log(gen.next().value);  //7
console.log(gen.next().value);  //8
```
从上面的代码可以看出，yield*后面如果跟的是一个Array或String，每执行一次next()函数，将会迭代Array或String中的一个元素，而如果是普通的yield则直接返回这个对象。

## arguments
```
function* GenFunc(){
    yield arguments;
    yield* arguments;
}

var gen = GenFunc(1, 2);
console.log(gen.next().value);  //{ '0': 1, '1': 2}
console.log(gen.next().value);  //1
console.log(gen.next().value);  //2
```
arguments的情况和Array或String情况一样，说明yield*后面如果跟的是一个可迭代对象，每执行一次next()函数，将会迭代一次这个对象。

## Generator
```
function* Gen1(){
    yield 2;
    yield 3;
    return 'liuxin';
}
function* Gen2(){
    yield 1;
    var a = yield* Gen1();
    console.log(a);   //打印'liuxin'
    yield 4;
}


var gen2 = Gen2();
console.log(gen2.next().value);  //1
console.log(gen2.next().value);  //2
console.log(gen2.next().value);  //3
console.log(gen2.next().value);  //4
```
yield*后面如果跟的是一个Generator函数，那么便会执行这个Generator函数，同时yield*这个表达式的值就是这个Generator函数的返回值。

## Object
```
function* GenFunc(){
    yield {a: '1', b: '2'};
    yield* {a: '1', b: '2'};
}

var gen = GenFunc();
console.log(gen.next().value);  //{ a: '1', b: '2'}
console.log(gen.next().value);  //1
console.log(gen.next().value);  //2
```
情况和arguments的情况一样。

## 总结
根据上面的所有代码可以知道，yield*后面接受一个iterable object作为参数，然后去迭代(iterate)这个迭代器（iterable object)，同时yield*本身这个表达式的值就是迭代器迭代完成时的返回值，以及yield*可以用来在一个generator函数里“执行”另一个generator函数，并可取得其返回值。

参考：http://taobaofed.org/blog/2015/11/19/yield-and-delegating-yield/



