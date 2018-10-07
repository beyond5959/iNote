title: JavaScript中的this
date: 2015-11-16 10:17:01
tags: [JavaScript, this]
---

JavaScript中的this在运行期进行绑定，这使得JavaScript中的this关键字具备多重含义。
<!-- more -->
JavaScript中的this到底指向什么？可以用下面一张图来解释：

![this的指向](http://nnblog-storage.b0.upaiyun.com/img/this.jpg!watermark1.0)

## 指向全局对象的例子

```
var point = {
	x: 0,
	y: 0,
	moveTo: function(x, y){
		//内部函数
		var moveX = function(x){
			this.x = x;   //this指向什么？window或global
		}；
		//内部函数
		var moveY = function(y){
			this.y = y;    //this指向什么？window或global
		}；
		moveX(x);
		moveY(y);
	}
};
point.moveTo(1, 1);
point.x;  //=>0
point.y;  //=>0
x;  //=>1
y;   //=>1
```
在上面的代码中，moveX(x)函数调用既不是用new进行调用，也不是用dot(.)进行调用，所以this指向window。

## 构造函数的例子

```
function Point(x, y){
	this.x = x; // this?
	this.y = y; // this?
}
var np = new Point(1, 1);
np.x; //1
var p = Point(2, 2);
p.x;  //error, p是一个空对象undefined
window.x;  //2
```

上面的代码中var np = new Point(1, 1)是用new进行调用，所以this指向创建的对象np，所以np.x就为1。
var p = Point(2, 2)函数调用既不是用new进行调用，也不是用dot(.)进行调用，所以this指向window。               

## call和apply进行调用的例子

```
function Point(x, y){
	this.x = x;
	this.y = y;
	this.moveTo = function(x, y){
		this.x = x;
		this.y = y;
		}；
}
var p1 = new Point(0, 0);
var p2 = {x: 0, y: 0};
p1.moveTo.apply(p2, [10, 10]);  //apply实际上为p2.moveTo(10, 10)
p2.x   //10
```

apply和call这两个方法可以改变函数执行的上下文，即改变this绑定的对象。p1.moveTo.apply(p2, [10,10])实际上是p2.moveTo(10, 10)。那么p2.moveTo(10, 10)可解释为不是new调用也而是dot(.)调用，所以this绑定的对象是p2。
