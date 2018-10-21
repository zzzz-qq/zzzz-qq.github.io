---
title: 《CSAPP》实验一：位操作
date: 2018-10-19 00:11:19
mathjax: true
tags:
    - CSAPP
    - C语言
    - 位操作
---
&emsp;&emsp;[《CSAPP》](https://book.douban.com/subject/26912767/)号称程序员圣经，虽然中文译名为《深入理解计算机系统》，但其实没那么“深”，只是覆盖面很广，一般用作计算机专业大一导论课的教科书。早就听闻书上配套的实验十分经典，这次重温新版（第三版），打算把所有的实验都做一下，也写个系列博文，好记录实验过程。实验可以在书本配套网站[CSAPP: Lab Assignments](http://csapp.cs.cmu.edu/3e/labs.html)下载，这篇从第一个实验 —— 位操作开始。<!-- more -->
#### 概述
&emsp;&emsp;本实验是第二章《信息的表示与处理》的配套实验，要求使用一个高度限制的C语言子集实现一些特定的逻辑，整数，浮点数的函数。延用第一章的说法，信息就是位加上下文，计算机系统中所有的信息都是由一串比特（或者说一串二进制数字）表示的，第二章就讲了C语言中整数与浮点数的编码方式，即典型地，计算机是如何用一串比特来表示整数与浮点数的：

* 无符号整数：直接二进制编码
* 有符号整数：二进制补码，最高位为负权
* 浮点数：[IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)

&emsp;&emsp;同样从内存里取出4个字节 $0x80000000$ ，把它当无符号整数看，就是 $2147483648$；把它当有符号整数看，就是 $-2147483648$；把它当单精度浮点数看，就是 $-0$。所谓上下文，就是解读这串比特的方式，横看成岭侧成峰。值得注意的是，尽管在几乎所有系统上，C语言整数与浮点数都是这么编码的，但C语言标准本身并没有这样规定，不知道有生之年能不能遇上非主流的编码方式。
&emsp;&emsp;如果没有完全掌握这些数字的编码方式以及C语言的位操作，是一定无法完成实验一的。实验一好就好在会让你反复回忆这些基础知识，深入细节之中，做完实验后想忘都忘不了：）
#### 前提
&emsp;&emsp;尽管有C语言有标准，但Undefined Behavior还是太多，尤其是深入底层进行位操作的情况下，因此实验预设： 有符号整数使用32位二进制补码编码； 右移操作为算术位移，高位补符号位。实验还要求：不能使用宏；整数操作不能使用大于0xFF的常量。下面就逐个函数记录实验过程了。
#### bitAnd
&emsp;&emsp;用`~`和`|`实现`&`，有公式很简单，但记不住，用韦恩图辅助思考：全集表示所有位都为1，`x`与`y`分别表示特定位置为1的子集，想象一下`~`，`|`和`&`的韦恩图，一下子就推出公式来了。
```c
/*
 * bitAnd - x&y using only ~ and |
 *   Example: bitAnd(6, 5) = 4
 *   Legal ops: ~ |
 *   Max ops: 8
 *   Rating: 1
 */
int bitAnd(int x, int y) {
    return ~(~x | ~y);
}
```
#### getByte
&emsp;&emsp;`x`右移 $n \* 8$ 位，取最后一个字节即可，利用了`n * 8 == n << 3`。
```c
/*
 * getByte - Extract byte n from word x
 *   Bytes numbered from 0 (LSB) to 3 (MSB)
 *   Examples: getByte(0x12345678,1) = 0x56
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 6
 *   Rating: 2
 */
int getByte(int x, int n) {
    return (x >> (n << 3)) & 0xFF;
}
```
#### logicalShift
&emsp;&emsp;实验预设了右移为算术位移，那么对`x`右移`n`位再用掩码将高位补的`n`位置0即可。
```c
/*
 * logicalShift - shift x to the right by n, using a logical shift
 *   Can assume that 0 <= n <= 31
 *   Examples: logicalShift(0x87654321,4) = 0x08765432
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 20
 *   Rating: 3
 */
int logicalShift(int x, int n) {
    int mask = ~(((1 << 31) >> n) << 1);
    return (x >> n) & mask;
}
```
#### bitCount
&emsp;&emsp;这题想了很久，正常的想法是将`x`一位一位地右移，用掩码1取最低位，再求和，然而操作符数量超标:D 然后想到，用`x & 1`去检查`x`最后一位是否是1比较亏，可以用`x & 0x00010001`，这样可以一次检查两位，最后将前后16位的结果汇总即可，然而操作符数量还是超标:D最终将`x`分了8组，`x & 0x11111111`，每次检查8位，用了38个操作符，终于达标。这是所有题目中用的操作符数量最多的一题了。
```c
/*
 * bitCount - returns count of number of 1's in word
 *   Examples: bitCount(5) = 2, bitCount(7) = 3
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 40
 *   Rating: 4
 */
int bitCount(int x) {
    int mask = 0x11 + (0x11 << 8) + (0x11 << 16) + (0x11 << 24);

    int count = (x & mask) + ((x >> 1) & mask) +
        ((x >> 2) & mask) + ((x >> 3) & mask);

    return (count & 7) + ((count >> 4) & 7) + ((count >> 8) & 7) +
        ((count >> 12) & 7) + ((count >> 16) & 7) + ((count >> 20) & 7) +
        ((count >> 24) & 7) + ((count >> 28) & 7);
}
```
#### bang
&emsp;&emsp;一开始想在0上面作文章，毕竟只有`bang(0) = 1`，但此路不通。`|`操作，二分法，逐渐把高位的1收集到低位，如`x = x | (x >> 16)`，如果高位的16位有1的话，就会被收集到低位的16位上，依此二分，收集到最后一位，刚好12个操作符。
```c
/*
 * bang - Compute !x without using !
 *   Examples: bang(3) = 0, bang(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12`
 *   Rating: 4
 */
int bang(int x) {
    x = x | (x >> 16);
    x = x | (x >> 8);
    x = x | (x >> 4);
    x = x | (x >> 2);
    x = x | (x >> 1);
    return ~x & 1;
}
```
#### tmin
&emsp;&emsp;最简单的一题，要熟悉二进制补码。
```c
/*
 * tmin - return minimum two's complement integer
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
int tmin(void) {
    return 1 << 31;
}
```
#### fitsBits
&emsp;&emsp;若`x`非负，考虑到`n`位二进制补码能表示的最大非负数为 $0b0111...111 $ （共`n-1`个1），用掩码将`x`低位的`n-1`位置0，检查高位的`32 - (n - 1)`位是否为0即可。若`x`为负，先将其转为非负数`~x`，编码`~x`必需的位数与编码`x`的是相同的。
```c
/*
 * fitsBits - return 1 if x can be represented as an
 *  n-bit, two's complement integer.
 *   1 <= n <= 32
 *   Examples: fitsBits(5,3) = 0, fitsBits(-4,3) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int fitsBits(int x, int n) {
    int minusOne = ~0;
    int mask = minusOne << (n + minusOne);
    return !((x ^ (x >> 31)) & mask);
}
```
#### divpwr2
&emsp;&emsp;`x >> n`即为$\lfloor x / 2^n \rfloor$，结果是向下取整的，但题目要求向0取整，若`x`非负向下取整即是向0取整没有问题，若`x`为负，需要向`x`加上一个偏移值$2^n - 1$，使得`x >> n`向上取整。
```c
/*
 * divpwr2 - Compute x/(2^n), for 0 <= n <= 30
 *  Round toward zero
 *   Examples: divpwr2(15,1) = 7, divpwr2(-33,4) = -2
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int divpwr2(int x, int n) {
    int signBit = (x >> 31) & 1;
    int bias = (signBit << n) + (~signBit + 1);
    return (x + bias) >> n;
}
```
#### negate
&emsp;&emsp;n位二进制补码的值域是$[-2^{n-1},\ 2^{n-1} - 1]$，并不关于0对称，因此当`x`为最小值时`-x`是它自己。
```c
/*
 * negate - return -x
 *   Example: negate(1) = -1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
int negate(int x) {
  return ~x + 1;
}
```
#### isPositive
&emsp;&emsp;正数的符号位为0，0的符号位也是0，是特殊情况。
```c
/*
 * isPositive - return 1 if x > 0, return 0 otherwise
 *   Example: isPositive(-1) = 0.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 8
 *   Rating: 3
 */
int isPositive(int x) {
    return (!!x) & (!(x >> 31));
}
```
#### isLessOrEqual
&emsp;&emsp;`isLessOrEqual`等价于`!isGreater`，实现`isGreater`简单点：若`x` `y`异号，则`x`必须非负`y`必须为负；若`x` `y` 同号，`x - y`不会溢出，必有`x - y > 0`，即`x - y - 1 >= 0`，即`x + ~y >= 0`，检查`x + ~y`的符号位即可。
```c
/*
 * isLessOrEqual - if x <= y  then return 1, else return 0
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
int isLessOrEqual(int x, int y) {
    int xSign = x >> 31;
    int ySign = y >> 31;

    int hasSameSign = !(xSign ^ ySign);
    int diffSign = (x + ~y) >> 31;

    int isXPosYNeg = (!xSign) & ySign;
    int isGreater = isXPosYNeg | (hasSameSign & !diffSign);

    return !isGreater;
}
```
#### ilog2
&emsp;&emsp;这道题允许90个操作符，是所有题目对操作符数量最宽松的了。`ilog2`的实质是求`x`最高位的1的索引，若`x`高位的16位有1，则不用管低位的16位；若`x`高位的8位有1，则不用管低位的24位，依次类推。实现起来还是十分巧妙的:)
```c
/*
 * ilog2 - return floor(log base 2 of x), where x > 0
 *   Example: ilog2(16) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 90
 *   Rating: 4
 */
int ilog2(int x) {
    int high16, high8, high4, high2, high1;

    high16 = (!!(x >> 16)) << 4;
    x = x >> high16;

    high8 = (!!(x >> 8)) << 3;
    x = x >> high8;

    high4 = (!!(x >> 4) << 2);
    x = x >> high4;

    high2 = (!!(x >> 2) << 1);
    x = x >> high2;

    high1 = !!(x >> 1);
    return high1 + high2 + high4 + high8 + high16;
}
```
#### float_neg
&emsp;&emsp;终于到浮点数了，浮点数的题对操作符要求宽松一点，还可以用循环跟判断语句。第一题，只要对IEEE754熟悉就行了。
```c
/*
 * float_neg - Return bit-level equivalent of expression -f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representations of
 *   single-precision floating point values.
 *   When argument is NaN, return argument.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 10
 *   Rating: 2
 */
unsigned float_neg(unsigned uf) {
    int isNaN = (((uf >> 23) & 0xFF) == 0xFF) && (uf << 9);
    return isNaN ? uf : ((1 << 31) ^ uf);
}
```
#### float_i2f
&emsp;&emsp;没什么技巧，十分暴力。从符号位，阶码，尾数，舍入，一个一个来。注意，`float(x)`是向偶数取整的。
```c
/*
 * float_i2f - Return bit-level equivalent of expression (float) x
 *   Result is returned as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point values.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_i2f(int x) {
    unsigned sign = x & (1 << 31);
    unsigned exp = 0;
    unsigned frac = 0;
    unsigned round = 0;

    unsigned absX = sign ? (~x + 1) : x;
    unsigned tmp = absX;
    while ((tmp = tmp >> 1))
        ++exp;

    frac = absX << (31 - exp) << 1;
    round = frac << 23 >> 23;
    frac = frac >> 9;

    if (round > 0x100) round = 1;
    else if (round < 0x100) round = 0;
    else round = frac & 1;

    return x ? (sign | ((exp + 0x7F) << 23) | frac) + round : 0;
}
```
#### float_twice
&emsp;&emsp;还是很暴力，按照浮点数分类一个一个来：特殊值，直接返回；规范化的浮点数，阶码加1；非规范化的，左移一位并保持符号位不变。
```c
/*
 * float_twice - Return bit-level equivalent of expression 2*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_twice(unsigned uf) {
    unsigned sign = 1 << 31;
    unsigned isNormalized = uf << 1 >> 24;
    unsigned isSpecial = isNormalized == 0xFF;

    if (isSpecial || uf == 0 || uf == sign)
        return uf;

    if (isNormalized)
        return uf + (1 << 23);

    return (uf << 1) | (uf & sign);
}
```