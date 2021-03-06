---
title: 由left-pad扯到JS中的位运算
date: 2017-01-07 18:04:12
tags: [JavaScript]
---

这个话题的由来是2016年3月份的时候 [NPM](https://www.npmjs.com/) 社区发生了‘left-pad’事件，不久后社区就有人发布了用来补救的，也是现在大家能用到的 [left-pad](https://www.npmjs.com/package/left-pad) 库。

<!-- more -->

最开始这个库的代码是这样的。

```javascript
module.exports = leftpad;
          
function leftpad (str, len, ch) {
  str = String(str);
  
  var i = -1;
  
  if (!ch && ch !== 0) ch = ' ';
  
  len = len - str.length;
  
  while (++i < len) {
    str = ch + str;
  }
  
  return str;
}
```

我第一次看到这段代码的时候，没看出什么毛病，觉得清晰明了。后来刷微博的时候[@左耳朵耗子](http://weibo.com/p/1005051401880315/home?from=page_100505&mod=TAB#place)老师指出了这段代码可以写得更有效率些，于是他就贴出了自己写的版本并给 left-pad 提了 PR，代码如下：

```javascript
module.exports = leftpad;
     
function leftpad (str, len, ch) {
  //convert the `str` to String
  str = str +'';
     
  //needn't to pad
  len = len - str.length;
  if (len <= 0) return str;
     
  //convert the `ch` to String
  if (!ch && ch !== 0) ch = ' ';
  ch = ch + '';
     
  var pad = '';
  while (true) {
    if (len & 1) pad += ch;
    len >>= 1;
    if (len) ch += ch;
    else break;
  }
  return pad + str;
}
```

我当时看到他的这段代码的里面的 `&`和`>>`运算符的时候一下有点懵了，只知道这是位运算里面的‘按位与’和‘右移’运算，但是完全不知道为什么这样写就能提高效率。于是就想着去了解位运算的实质和使用场景。

在了解位运算之前，我们先必须了解一下什么是原码、反码和补码以及二进制与十进制的转换。

### 原码、补码和反码

#### 原码
一个数在计算机中是以二进制的形式存在的，其中第一位存放符号, 正数为0, 负数为1。原码就是用第一位存放符号的二进制数值。例如2的原码为00000010，-2的原码为10000010。

#### 反码
正数的反码是其本身。负数的反码是在其原码的基础上，符号位不变，其余各位取反，即0变1，1变0。
```javascript
[+3]=[00000011]原=[00000011]反
[-3]=[10000011]原=[11111100]反
```
可见如果一个反码表示的是负数，并不能直观的看出它的数值，通常要将其转换成原码再计算。

#### 补码
正数的补码是其本身。负数的补码是在其原码的基础上，符号位不变，其余各位取反，最后+1。（即负数的补码为在其反码的基础上+1）。
```javascript
[+3]=[00000011]原=[00000011]反=[00000011]补
[-3]=[10000011]原=[11111100]反=[11111101]补
```
可见对于负数，补码的表示方式也是让人无法直观看出其数值的，通常也需要转换成原码再计算。

### 二进制与十进制的转换
二进制与十进制的区别在于数运算时是逢几进一位。二进制是逢2进一位，十进制也就是我们常用的0-9是逢10进一位

#### 正整数的十进制转二进制
正整数的十进制转二进制的方法为将一个十进制数除以2，得到的商再除以2，以此类推直到商等于1或0时为止，倒取除得的余数，即为转换所得的二进制数的结果。

例如把52换算成二进制数，计算过程如下图：
![10to2](http://nnblog-storage.b0.upaiyun.com/img/2to10.gif)
52除以2得到的余数依次为：0、0、1、0、1、1，倒序排列，所以52对应的二进制数就是110100。

#### 负整数的十进制转二进制
负整数的十进制转二进制为将该负整数对应的正整数先转换成二进制，然后对其“取反”，再对取反后的结果+1。即负整数采用其二进制补码的形式存储。
至于负数为什么要用二进制补码的形式存储，可参考一篇阮一峰的文章《[关于2的补码](http://www.ruanyifeng.com/blog/2009/08/twos_complement.html)》。
例如 -52 的原码为 10110100，其反码为 11001011，其补码为 11001100。所以 -52 转换为二进制后为 11001100。

#### 十进制小数转二进制
十进制小数转二进制的方法为“乘2取整”，对十进制小数乘2得到的整数部分和小数部分，整数部分即是相应的二进制数码，再用2乘小数部分(之前乘后得到新的小数部分)，又得到整数和小数部分。
如此不断重复，直到小数部分为0或达到精度要求为止。第一次所得到为最高位，最后一次得到为最低位。
```
如:0.25的二进制
0.25*2=0.5 取整是0
0.5*2=1.0    取整是1
即0.25的二进制为 0.01 ( 第一次所得到为最高位,最后一次得到为最低位)
   
0.8125的二进制
0.8125*2=1.625   取整是1
0.625*2=1.25     取整是1
0.25*2=0.5       取整是0
0.5*2=1.0        取整是1
即0.8125的二进制是0.1101
```

#### 二进制转十进制
从最后一位开始算，依次列为第0、1、2...位，第n位的数（0或1）乘以2的n次方，将得到的结果相加就是得到的十进制数。
例如二进制为110的数，将其转为十进制的过程如下
![2to10](http://nnblog-storage.b0.upaiyun.com/img/2to10.png)
个位数 0 与 2º 相乘：0 × 2º = 0
十位数 1 与 2¹ 相乘：1 × 2¹ = 2
百位数 1 与 2² 相乘：1 × 2² = 4
将得到的结果相加：0+2+4=6
所以二进制 110 转换为十进制后的数值为 6。

小数二进制用数值乘以2的负幂次然后相加。

### JavaScript 中的位运算
在 ECMAScript 中按位操作符会将其操作数转成**补码形式的有符号32位整数。**下面是[ECMAScript 规格](http://www.ecma-international.org/ecma-262/6.0/#sec-binary-bitwise-operators-runtime-semantics-evaluation)中对于位运算的执行过程的表述：
```
The production A : A @ B, where @ is one of the bitwise operators in the productions above, is evaluated as follows:
1. Let lref be the result of evaluating A.
2. Let lval be GetValue(lref).
3. ReturnIfAbrupt(lval).
4. Let rref be the result of evaluating B.
5. Let rval be GetValue(rref).
6. ReturnIfAbrupt(rval).
7. Let lnum be ToInt32(lval).
8. ReturnIfAbrupt(lnum).
9. Let rnum be ToInt32(rval).
10. ReturnIfAbrupt(rnum).
11. Return the result of applying the bitwise operator @ to lnum and rnum. The result is a signed 32 bit integer.
```
需要注意的是第七步和第九步，根据 ES 的标准，超过32位的整数会被截断，而小数部分则会被直接舍弃。所以由此可以知道，在 JS 中，当位运算中有操作数大于或等于2³²时，就会出现意想不到的结果。

JavaScript 中的位运算有：`&（按位与）`、`|（按位或）`、`~（取反）`、`^（按位异或）`、`<<（左移）`、`>>（有符号右移）`和`>>>（无符号右移）`。

#### &按位与
对每一个比特位执行与（AND）操作。只有 a 和 b 都是 1 时，a & b 才是 1。
例如：9(base 10) & 14(base 10) = 1001(base2) & 1110(base 2) = 1000(base 2) = 8(base 10)

因为当只有 a 和 b 都是 1 时，a&b才等于1，所以任一数值 x 与0（二进制的每一位都是0）按位与操作，其结果都为0。将任一数值 x 与 -1（二进制的每一位都是1）按位与操作，其结果都为 x。
利用 & 运算的特点，我们可以用以简单的判断奇偶数，公式：
```javascript
(n & 1) === 0 //true 为偶数，false 为奇数。
```
因为 1 的二进制只有最后一位为1，其余位都是0，所以其判断奇偶的实质是判断二进制数最后一位是 0 还是 1。奇数的二进制最后一位是 1，偶数是0。

当然还可以利用 JS 在做位运算时会舍弃掉小数部分的特性来做向下取整的运算，因为当 x 为整数时有 `x&-1=x`，所以当 x 为小数时有 `x&-1===Math.floor(x)`。

#### |按位或
对每一个比特位执行或（OR）操作。如果 a 或 b 为 1，则 a | b 结果为 1。
例如：9(base 10) | 14(base 10) = 1001(base2) | 1110(base 2) = 1111(base 2) = 15(base 10)

因为只要 a 或 b 其中一个是 1 时，a|b就等于1，所以任一数值 x 与-1（二进制的每一位都是1）按位与操作，其结果都为-1。将任一数值 x 与 0（二进制的每一位都是0）按位与操作，其结果都为 x。

同样，按位或也可以做向下取整运算，因为当 x 为整数时有 `x|0=x`，所以当 x 为小数时有 `x|0===Math.floor(x)`。

#### ~取反
对每一个比特位执行非（NOT）操作。~a 结果为 a 的反转（即反码）。
```
9 (base 10)  = 00000000000000000000000000001001 (base 2)
               --------------------------------
~9 (base 10) = 11111111111111111111111111110110 (base 2) = -10 (base 10)
```
负数的二进制转化为十进制的规则是，符号位不变，其他位取反后加 1。

对任一数值 x 进行按位非操作的结果为 -(x + 1)。~~x === x。

同样，取反也可以做向下取整运算，因为当 x 为整数时有 `~~x===x`，所以当 x 为小数时有 `~~x===Math.floor(x)`。

#### ^按位异或
对每一对比特位执行异或（XOR）操作。当 a 和 b 不相同时，a ^ b 的结果为 1。
例如：9(base 10) ^ 14(base 10) = 1001(base2) ^ 1110(base 2) = 0111(base 2) = 7(base 10)

将任一数值 x 与 0 进行异或操作，其结果为 x。将任一数值 x 与 -1 进行异或操作，其结果为 ~x，即 x^-1=~x。
同样，按位异或也可以做向下取整运算，因为当 x 为整数时有 `(x^0)===x`，所以当 x 为小数时有 `(x^0)===Math.floor(x)`。

#### <<左移运算
它把数字中的所有数位向左移动指定的数量，向左被移出的位被丢弃，右侧用 0 补充。
例如，把数字 2（等于二进制中的 10）左移 5 位，结果为 64（等于二进制中的 1000000）：
```javascript
var iOld = 2;		//等于二进制 10
var iNew = iOld << 5;	//等于二进制 1000000 十进制 64
```
因为二进制10转换成十进制的过程为 1×2¹+0×2º，在运算中2的指数与位置数相对应，当左移五位后就变成了 1×2¹⁺⁵+0×2º⁺⁵= 1×2¹×2⁵+0×2º×2⁵ = (1×2¹+0×2º)×2⁵。所以由此可以看出当2左移五位就变成了 2×2⁵=64。
所以有一个数左移 n 为，即为这个数乘以2的n次方。`x<<n === x*2ⁿ`。
同样，左移运算也可以做向下取整运算，因为当 x 为整数时有 `(x<<0)===x`，所以当 x 为小数时有 `(x<<0)===Math.floor(x)`。

#### >>有符号右移运算
它把 32 位数字中的所有数位整体右移，同时保留该数的符号（正号或负号）。有符号右移运算符恰好与左移运算相反。例如，把 64 右移 5 位，将变为 2。
因为有符号右移运算符与左移运算相反，所以有一个数左移 n 为，即为这个数除以2的n次方。`x<<n === x/2ⁿ`。
同样，有符号右移运算也可以做向下取整运算，因为当 x 为整数时有 `(x>>0)===x`，所以当 x 为小数时有 `(x>>0)===Math.floor(x)`。

#### >>>无符号右移运算
它将无符号 32 位数的所有数位整体右移。对于正数，无符号右移运算的结果与有符号右移运算一样，而负数则被作为正数来处理。
```javascript
-9 (base 10): 11111111111111111111111111110111 (base 2)
                    --------------------------------
-9 >>> 2 (base 10): 00111111111111111111111111111101 (base 2) = 1073741821 (base 10)
```
根据无符号右移的正数右移与有符号右移运算一样，而负数的无符号右移一定为非负的特征，可以用来判断数字的正负，如下：
```javascript
function isPos(n) {
  return (n === (n >>> 0)) ? true : false;  
}
    
isPos(-1); // false
isPos(1); // true
```

### 总结
根据 JS 的位运算，可以得出如下信息：
1、所有的位运算都可以对小数取底。
2、对于按位与`&`，可以用 `(n & 1) === 0 //true 为偶数，false 为奇数。`来判断奇偶。用`x&-1===Math.floor(x)`来向下取底。
3、对于按位或`|`，可以用`x|0===Math.floor(x)`来向下取底。
4、对于取反运算`~`，可以用`~~x===Math.floor(x)`来向下取底。
5、对于异或运算`^`，可以用`(x^0)===Math.floor(x)`来向下取底。
6、对于左移运算`<<`，可以`x<<n === x*2ⁿ`来求2的n次方，用`x<<0===Math.floor(x)`来向下取底。
7、对于有符号右移运算`>>`，可以`x<<n === x/2ⁿ`求一个数字的 N 等分，用`x>>0===Math.floor(x)`来向下取底。
8、对于无符号右移运算`>>>`，可以`(n === (n >>> 0)) ? true : false;`来判断数字正负，用`x>>>0===Math.floor(x)`来向下取底。

用移位运算来替代普通算术能获得更高的效率。移位运算翻译成机器码的长度更短，执行更快，需要的硬件开销更小。
