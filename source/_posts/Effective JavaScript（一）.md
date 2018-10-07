title: Effective JavaScript（一）
date: 2016-05-14 20:00:38
tags: [JavaScript]
---
最近在读《Effective JavaScript》这本书，书中讲解了很多JS的语言细节和一些最佳实践，有一些是我平时使用时没有注意的，也有一些是自己踩过的坑，下面就列举一些我所留意的知识点。
<!-- more -->
## Number
JS中的数字都是双精度的浮点数。JS中的整数仅仅是双精度浮点数的一个子集，而不是一个单独的数据类型。<br />
JS的浮点数在运算的时候存在精度陷阱。
```
  0.1 + 0.2 // 0.30000000000000004
```
因为浮点数运算只能产生近似的结果，四舍五入到最接近的可表示的实数，当执行一系列的运算，随着舍入误差的积累，运算结果会越来越不精确。<br />
若想要使*0.1+0.2*等于0.3，可以使用*Number((0.1+0.2).toFixed(1))* 来实现，注意toFixed()生成的是字符串，所以必须用Number来转换成数字。

NaN是JS中的一个特殊数字，它是唯一一个不等于本身的数字。
```
var a = Number(b); // a为NaN, 因为b未定义
a == a; // false
a != a; // true
```

## JS中的7个假值
JS中只有7个布尔类型为false的值，它们分别是`false(Boolean)`、`0(Number)`、`-0(Nubmer)`、`""(String)`、`null(Object)`、`NaN(Number)`、`undefined`。

`注意，空对象{}和空数组[]它们的布尔类型为true。`

## toString()和String()
这两个方法都可以将一个值转换成一个字符串。
#### toStirng()
  - 大多数值都有toString()方法，null和undefined是没有的。
  - 对应字符串型的值也可以使用toString()方法，它会返回该字符串的一个副本。
  - toString()方法可以传递一个参数，表示数值的基数。（二进制，十进制）
  ```
  var t = 8;
  t.toString(2); // '1000', 对于非数字类型设置toString()的参数是无效的。
  ```

#### String()
任何值都可以使用String()方法。首先如果值有toString()方法，那么则使用该方法。其次，如果没有toString()方法，那就是null返回'null'，undefined返回'undefined'。

## encodeURI和encodeURIComponent
encodeURI和encodeURIComponent是把字符编码成UTF-8。<br />
encodeURI方法不会对ASCII字母、数字、~!@#$%&*()=:/,;?+-_这些字符编码。<br />
encodeURIComponent方法不会对ASCII字母、数字、~!*()'这些字符编码。<br />
所以encodeURIComponent比encodeURI的编码范围更广。<br />
它们对应的解码方法分别是decodeURI和decodeURIComponent。<br />

## 闭包
闭包存储的是外部变量的引用而不是值，并能读写这些值。<br />
下面这段代码输出什么？
```
function wrapElements(a) {
  var result = [], i, n;
  for (i = 0; len = a.length; i < len; ++i) {
    result[i] = function() { return a[i]; };  // ①
  }
  return result;
}

var wrapped = wrapElements([10, 20, 30, 40, 50]);
var f = wrapped[0];
f(); // ?
```
我刚开始以为是50，但实际是undefined。<br />
因为闭包存储的是外部变量的引用，所以注释①那行代码中a[i]中的i其实是对i的引用，此时i的值为5，a[5]没有值，所以为undefined。
