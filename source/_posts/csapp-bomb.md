---
title: 《CSAPP》实验二：二进制炸弹
date: 2018-12-05 00:08:36
tags:
    - CSAPP
    - GDB
---

&emsp;&emsp;二进制炸弹是第三章《程序的机器级表示》的配套实验，这章主要介绍了x64汇编，包括：操作数的表示方式，数据传送指令，算术和逻辑指令，控制流跳转指令，过程（procedure）的实现与运行时栈帧，C语言中的数组，struct，union以及浮点数的汇编表示等。通过这章的学习，对C有了更深的理解，可以看出，C与汇编代码的相似度很高，称之为高级汇编也不为过。<!-- more -->

&emsp;&emsp;这个实验提供了一个 Linux/x86-64 二进制程序（下载地址：[CSAPP: Labs](http://csapp.cs.cmu.edu/3e/labs.html)），即所谓的“二进制炸弹”。执行这个程序，它会要求你逐个输入6个字符串，只要输错了一个，“炸弹”就会被引爆。实验要求我们利用GDB对这个“炸弹”进行逆向工程，找到6个正确的字符串。整个实验十分有趣，寓教于乐，完成之后很有成就感。实验的基本思路如下：

* 在各个检查输入字符串的地方设断点
* 先随便输入字符串，执行到断点处
* 反汇编，找到正确字符串，保存答案，去掉对应的断点，继续

&emsp;&emsp;GDB的各种操作，下载一张[速查表](https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf)，反复用就熟悉了。实验也提供了“炸弹”的`main`函数源码，可以看出输入的字符串分别由6个函数检查，分别是 `phase_1`，`phase_2`，...，`phase_6`。在`phase_1`设好断点，实验就开始啦：

```bash
$ gdb bomb
(gdb) break phase_1
(gdb) run
```

#### phase_1
`(gdb) disas ` 反汇编代码如下：
```
=> 0x0000000000400ee0 <+0>:	sub    $0x8,%rsp
   0x0000000000400ee4 <+4>:	mov    $0x402400,%esi
   0x0000000000400ee9 <+9>:	callq  0x401338 <strings_not_equal>
   0x0000000000400eee <+14>:	test   %eax,%eax
   0x0000000000400ef0 <+16>:	je     0x400ef7 <phase_1+23>
   0x0000000000400ef2 <+18>:	callq  0x40143a <explode_bomb>
   0x0000000000400ef7 <+23>:	add    $0x8,%rsp
   0x0000000000400efb <+27>:	retq
```

`phase_1`把两个字符串传给了`strings_not_equal`，若两个字符串不相等，炸弹就爆炸。输入的字符串是第一个参数`%rdi`，`$0x402400`是第二个参数，`(gdb) print (char*) 0x402400`，打印出来就是第一个字符串，第一题比较简单。

#### phase_2
```
Dump of assembler code for function phase_2:
=> 0x0000000000400efc <+0>:	push   %rbp
   0x0000000000400efd <+1>:	push   %rbx
   0x0000000000400efe <+2>:	sub    $0x28,%rsp
   0x0000000000400f02 <+6>:	mov    %rsp,%rsi
   0x0000000000400f05 <+9>:	callq  0x40145c <read_six_numbers>
   0x0000000000400f0a <+14>:	cmpl   $0x1,(%rsp)
   0x0000000000400f0e <+18>:	je     0x400f30 <phase_2+52>
   0x0000000000400f10 <+20>:	callq  0x40143a <explode_bomb>
   0x0000000000400f15 <+25>:	jmp    0x400f30 <phase_2+52>
   0x0000000000400f17 <+27>:	mov    -0x4(%rbx),%eax
   0x0000000000400f1a <+30>:	add    %eax,%eax
   0x0000000000400f1c <+32>:	cmp    %eax,(%rbx)
   0x0000000000400f1e <+34>:	je     0x400f25 <phase_2+41>
   0x0000000000400f20 <+36>:	callq  0x40143a <explode_bomb>
   0x0000000000400f25 <+41>:	add    $0x4,%rbx
   0x0000000000400f29 <+45>:	cmp    %rbp,%rbx
   0x0000000000400f2c <+48>:	jne    0x400f17 <phase_2+27>
   0x0000000000400f2e <+50>:	jmp    0x400f3c <phase_2+64>
   0x0000000000400f30 <+52>:	lea    0x4(%rsp),%rbx
   0x0000000000400f35 <+57>:	lea    0x18(%rsp),%rbp
   0x0000000000400f3a <+62>:	jmp    0x400f17 <phase_2+27>
   0x0000000000400f3c <+64>:	add    $0x28,%rsp
   0x0000000000400f40 <+68>:	pop    %rbx
   0x0000000000400f41 <+69>:	pop    %rbp
   0x0000000000400f42 <+70>:	retq
 ```

 翻译回C语言如下，第3行`sub $0x28,%rsp` 分配了一个数组，向前的跳转是循环，第二个字符串是个等比数列`"1 2 4 8 16 32"`：

```C
void phase_2(const char* input) {
    int rsp[6];
    read_six_numbers(rsp, nums);
    if (rsp[0] != 1)
        explode_bomb();

    int* rbx = rsp + 1;
    int* rpb = rsp + 6;
    do {
        int eax = rbx[-1];
        eax += eax;
        if (*rbx != eax)
            explode_bomb();

        rbx += 1;
    } while (rbx != rbp)
}
```


#### phase_3
```
Dump of assembler code for function phase_3:
=> 0x0000000000400f43 <+0>:	sub    $0x18,%rsp
   0x0000000000400f47 <+4>:	lea    0xc(%rsp),%rcx
   0x0000000000400f4c <+9>:	lea    0x8(%rsp),%rdx
   0x0000000000400f51 <+14>:	mov    $0x4025cf,%esi
   0x0000000000400f56 <+19>:	mov    $0x0,%eax
   0x0000000000400f5b <+24>:	callq  0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000400f60 <+29>:	cmp    $0x1,%eax
   0x0000000000400f63 <+32>:	jg     0x400f6a <phase_3+39>
   0x0000000000400f65 <+34>:	callq  0x40143a <explode_bomb>
   0x0000000000400f6a <+39>:	cmpl   $0x7,0x8(%rsp)
   0x0000000000400f6f <+44>:	ja     0x400fad <phase_3+106>
   0x0000000000400f71 <+46>:	mov    0x8(%rsp),%eax
   0x0000000000400f75 <+50>:	jmpq   *0x402470(,%rax,8)
   0x0000000000400f7c <+57>:	mov    $0xcf,%eax
   0x0000000000400f81 <+62>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400f83 <+64>:	mov    $0x2c3,%eax
   0x0000000000400f88 <+69>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400f8a <+71>:	mov    $0x100,%eax
   0x0000000000400f8f <+76>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400f91 <+78>:	mov    $0x185,%eax
   0x0000000000400f96 <+83>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400f98 <+85>:	mov    $0xce,%eax
   0x0000000000400f9d <+90>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400f9f <+92>:	mov    $0x2aa,%eax
   0x0000000000400fa4 <+97>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400fa6 <+99>:	mov    $0x147,%eax
   0x0000000000400fab <+104>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400fad <+106>:	callq  0x40143a <explode_bomb>
   0x0000000000400fb2 <+111>:	mov    $0x0,%eax
   0x0000000000400fb7 <+116>:	jmp    0x400fbe <phase_3+123>
   0x0000000000400fb9 <+118>:	mov    $0x137,%eax
   0x0000000000400fbe <+123>:	cmp    0xc(%rsp),%eax
   0x0000000000400fc2 <+127>:	je     0x400fc9 <phase_3+134>
   0x0000000000400fc4 <+129>:	callq  0x40143a <explode_bomb>
   0x0000000000400fc9 <+134>:	add    $0x18,%rsp
   0x0000000000400fcd <+138>:	retq
```
2 - 10：调用`sscanf`，格式地址在0x4025cf，值为`"%d %d"`，可见这一关要求输入两个整数。
11 - 12：要求第一个整数小于等于7。
13 - 14：典型的switch语句，根据第一个整数的值跳转，跳转表地址为0x402470。
15 - 34：根据跳转表设置第二个整数，答案不唯一，有8个，随便选个`"0 207"`。

`(gdb) x /8xg 0x402470`打印跳转表如下：
```
0x402470:	0x0000000000400f7c	0x0000000000400fb9
0x402480:	0x0000000000400f83	0x0000000000400f8a
0x402490:	0x0000000000400f91	0x0000000000400f98
0x4024a0:	0x0000000000400f9f	0x0000000000400fa6
```

#### phase_4
```
Dump of assembler code for function phase_4:
=> 0x000000000040100c <+0>:	sub    $0x18,%rsp
   0x0000000000401010 <+4>:	lea    0xc(%rsp),%rcx
   0x0000000000401015 <+9>:	lea    0x8(%rsp),%rdx
   0x000000000040101a <+14>:	mov    $0x4025cf,%esi
   0x000000000040101f <+19>:	mov    $0x0,%eax
   0x0000000000401024 <+24>:	callq  0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000401029 <+29>:	cmp    $0x2,%eax
   0x000000000040102c <+32>:	jne    0x401035 <phase_4+41>
   0x000000000040102e <+34>:	cmpl   $0xe,0x8(%rsp)
   0x0000000000401033 <+39>:	jbe    0x40103a <phase_4+46>
   0x0000000000401035 <+41>:	callq  0x40143a <explode_bomb>
   0x000000000040103a <+46>:	mov    $0xe,%edx
   0x000000000040103f <+51>:	mov    $0x0,%esi
   0x0000000000401044 <+56>:	mov    0x8(%rsp),%edi
   0x0000000000401048 <+60>:	callq  0x400fce <func4>
   0x000000000040104d <+65>:	test   %eax,%eax
   0x000000000040104f <+67>:	jne    0x401058 <phase_4+76>
   0x0000000000401051 <+69>:	cmpl   $0x0,0xc(%rsp)
   0x0000000000401056 <+74>:	je     0x40105d <phase_4+81>
   0x0000000000401058 <+76>:	callq  0x40143a <explode_bomb>
   0x000000000040105d <+81>:	add    $0x18,%rsp
   0x0000000000401061 <+85>:	retq
```
2 - 9：同`phase_3`，这一关也要求输入两个整数。
10 - 12：要求第一个整数小于等于 0xe。
13 - 16：调用`func4(第一个整数, 0, 0xe)`。
17 - 18：要求`func4`返回 0。
19 - 20：要求第二个整数为 0。

接着看`func4`：
```
Dump of assembler code for function func4:
   0x0000000000400fce <+0>:	sub    $0x8,%rsp
   0x0000000000400fd2 <+4>:	mov    %edx,%eax
   0x0000000000400fd4 <+6>:	sub    %esi,%eax
   0x0000000000400fd6 <+8>:	mov    %eax,%ecx
   0x0000000000400fd8 <+10>:	shr    $0x1f,%ecx
   0x0000000000400fdb <+13>:	add    %ecx,%eax
   0x0000000000400fdd <+15>:	sar    %eax
   0x0000000000400fdf <+17>:	lea    (%rax,%rsi,1),%ecx
   0x0000000000400fe2 <+20>:	cmp    %edi,%ecx
   0x0000000000400fe4 <+22>:	jle    0x400ff2 <func4+36>
   0x0000000000400fe6 <+24>:	lea    -0x1(%rcx),%edx
   0x0000000000400fe9 <+27>:	callq  0x400fce <func4>
   0x0000000000400fee <+32>:	add    %eax,%eax
   0x0000000000400ff0 <+34>:	jmp    0x401007 <func4+57>
   0x0000000000400ff2 <+36>:	mov    $0x0,%eax
   0x0000000000400ff7 <+41>:	cmp    %edi,%ecx
   0x0000000000400ff9 <+43>:	jge    0x401007 <func4+57>
   0x0000000000400ffb <+45>:	lea    0x1(%rcx),%esi
   0x0000000000400ffe <+48>:	callq  0x400fce <func4>
   0x0000000000401003 <+53>:	lea    0x1(%rax,%rax,1),%eax
   0x0000000000401007 <+57>:	add    $0x8,%rsp
   0x000000000040100b <+61>:	retq
```

翻译回C语言如下，注意`shr`是逻辑右移，`sar`是算术右移。要使`func4(rdi, 0, 0xe)`返回 0，必须`rcx == rdi`，很容易计算得出`rcx`为7，因此第一个整数为7，第四关答案为`"7 0"`。
```c
int func4(int rdi, int rsi, int rdx) {
    int rax = rdx - rsi;
    rax += ((rax >> 31) & 1);
    rax >>= 1;

    int rcx = rax + rsi;
    if (rcx > rdi) {
        rdx = rcx - 1;
        return 2 * func4(rdi, rsi, rdx);
    }

    rax = 0;
    if (rcx < rdi) {
        rsi = rcx + 1;
        return 2 * func4(rdi, rsi, rdx) + 1;
    }

    return rax;
}
```

#### phase_5
```
Dump of assembler code for function phase_5:
=> 0x0000000000401062 <+0>:	push   %rbx
   0x0000000000401063 <+1>:	sub    $0x20,%rsp
   0x0000000000401067 <+5>:	mov    %rdi,%rbx
   0x000000000040106a <+8>:	mov    %fs:0x28,%rax
   0x0000000000401073 <+17>:	mov    %rax,0x18(%rsp)
   0x0000000000401078 <+22>:	xor    %eax,%eax
   0x000000000040107a <+24>:	callq  0x40131b <string_length>
   0x000000000040107f <+29>:	cmp    $0x6,%eax
   0x0000000000401082 <+32>:	je     0x4010d2 <phase_5+112>
   0x0000000000401084 <+34>:	callq  0x40143a <explode_bomb>
   0x0000000000401089 <+39>:	jmp    0x4010d2 <phase_5+112>
   0x000000000040108b <+41>:	movzbl (%rbx,%rax,1),%ecx
   0x000000000040108f <+45>:	mov    %cl,(%rsp)
   0x0000000000401092 <+48>:	mov    (%rsp),%rdx
   0x0000000000401096 <+52>:	and    $0xf,%edx
   0x0000000000401099 <+55>:	movzbl 0x4024b0(%rdx),%edx
   0x00000000004010a0 <+62>:	mov    %dl,0x10(%rsp,%rax,1)
   0x00000000004010a4 <+66>:	add    $0x1,%rax
   0x00000000004010a8 <+70>:	cmp    $0x6,%rax
   0x00000000004010ac <+74>:	jne    0x40108b <phase_5+41>
   0x00000000004010ae <+76>:	movb   $0x0,0x16(%rsp)
   0x00000000004010b3 <+81>:	mov    $0x40245e,%esi
   0x00000000004010b8 <+86>:	lea    0x10(%rsp),%rdi
   0x00000000004010bd <+91>:	callq  0x401338 <strings_not_equal>
   0x00000000004010c2 <+96>:	test   %eax,%eax
   0x00000000004010c4 <+98>:	je     0x4010d9 <phase_5+119>
   0x00000000004010c6 <+100>:	callq  0x40143a <explode_bomb>
   0x00000000004010cb <+105>:	nopl   0x0(%rax,%rax,1)
   0x00000000004010d0 <+110>:	jmp    0x4010d9 <phase_5+119>
   0x00000000004010d2 <+112>:	mov    $0x0,%eax
   0x00000000004010d7 <+117>:	jmp    0x40108b <phase_5+41>
   0x00000000004010d9 <+119>:	mov    0x18(%rsp),%rax
   0x00000000004010de <+124>:	xor    %fs:0x28,%rax
   0x00000000004010e7 <+133>:	je     0x4010ee <phase_5+140>
   0x00000000004010e9 <+135>:	callq  0x400b30 <__stack_chk_fail@plt>
   0x00000000004010ee <+140>:	add    $0x20,%rsp
   0x00000000004010f2 <+144>:	pop    %rbx
   0x00000000004010f3 <+145>:	retq
```
3 - 4：分配一段栈空间（数组），保存输入的字符串到`%rbx`。
5 - 7：设置哨兵值，保护栈空间。
8 - 11：要求字符串长度为 6。
12 - 22：为一个循环，翻译回C如下，这段代码将输入的字符串做了个转换：
&emsp;&emsp;取字符的后4位作为索引，从预设的一个长字符串取转换后的字符。
23 - 26：比较转换后的字符串和预期的是否相等。

从预期的字符串以及转换规则反推回去，可得到第5关的答案是`"9?>567"`。

```c
const char* pattern =  // 第17行，print (char*) 0x4024b0
  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"; 
  
const char* input = "9?>567";

char transformed[7];  // 第3, 4行分配的数组
for (int rax = 0; rax != 6; ++rax) {
    int rcx = input[rax];
    int rdx = rcx & 0xf;
    transformed[rax] = (char)pattern[rdx];
}
transformed[6] = 0;  // 第22行
const char* expected = "flyers"; // 第23行，print (char*) 0x40245e
```

#### phase_6
这一关反汇编代码太长了，屏幕一页都放不下，最好分段分析。

```
Dump of assembler code for function phase_6:
=> 0x00000000004010f4 <+0>:	push   %r14
   0x00000000004010f6 <+2>:	push   %r13
   0x00000000004010f8 <+4>:	push   %r12
   0x00000000004010fa <+6>:	push   %rbp
   0x00000000004010fb <+7>:	push   %rbx
   0x00000000004010fc <+8>:	sub    $0x50,%rsp
   0x0000000000401100 <+12>:	mov    %rsp,%r13
   0x0000000000401103 <+15>:	mov    %rsp,%rsi
   0x0000000000401106 <+18>:	callq  0x40145c <read_six_numbers>
```
第一部分，分配了数组，读取6个数字，可见这一关要求我们输入6数字。
看到后面的反汇编有不止一个循环，可以分循环分析。

```
   0x000000000040110b <+23>:	mov    %rsp,%r14
   0x000000000040110e <+26>:	mov    $0x0,%r12d
   0x0000000000401114 <+32>:	mov    %r13,%rbp
   0x0000000000401117 <+35>:	mov    0x0(%r13),%eax
   0x000000000040111b <+39>:	sub    $0x1,%eax
   0x000000000040111e <+42>:	cmp    $0x5,%eax
   0x0000000000401121 <+45>:	jbe    0x401128 <phase_6+52>
   0x0000000000401123 <+47>:	callq  0x40143a <explode_bomb>
   0x0000000000401128 <+52>:	add    $0x1,%r12d
   0x000000000040112c <+56>:	cmp    $0x6,%r12d
   0x0000000000401130 <+60>:	je     0x401153 <phase_6+95>
   0x0000000000401132 <+62>:	mov    %r12d,%ebx
   0x0000000000401135 <+65>:	movslq %ebx,%rax
   0x0000000000401138 <+68>:	mov    (%rsp,%rax,4),%eax
   0x000000000040113b <+71>:	cmp    %eax,0x0(%rbp)
   0x000000000040113e <+74>:	jne    0x401145 <phase_6+81>
   0x0000000000401140 <+76>:	callq  0x40143a <explode_bomb>
   0x0000000000401145 <+81>:	add    $0x1,%ebx
   0x0000000000401148 <+84>:	cmp    $0x5,%ebx
   0x000000000040114b <+87>:	jle    0x401135 <phase_6+65>
   0x000000000040114d <+89>:	add    $0x4,%r13
   0x0000000000401151 <+93>:	jmp    0x401114 <phase_6+32>
```

上面这段包含了两个循环，翻译回C语言如下：
```c
int input[6];

for (int r12d = 0; r12d != 6; ++r12d) {
    int rax = input[r12d];
    if (rax - 1 > 5)
        explode_bomb();

    for (int rbx = r12d + 1; rbx <= 5; ++rbx) {
        if (rax == input[rbx])
            explode_bomb();
    }
}
```

这段代码检查了输入的6个数字，要求它们都小于等于6，互不相等，且要大于0，所以答案是`1 2 3 4 5 6`的排列。继续看下一部分：

```
   0x0000000000401153 <+95>:	lea    0x18(%rsp),%rsi
   0x0000000000401158 <+100>:	mov    %r14,%rax
   0x000000000040115b <+103>:	mov    $0x7,%ecx
   0x0000000000401160 <+108>:	mov    %ecx,%edx
   0x0000000000401162 <+110>:	sub    (%rax),%edx
   0x0000000000401164 <+112>:	mov    %edx,(%rax)
   0x0000000000401166 <+114>:	add    $0x4,%rax
   0x000000000040116a <+118>:	cmp    %rsi,%rax
   0x000000000040116d <+121>:	jne    0x401160 <phase_6+108>
   ```
   
   上面这部分代码对输入数组做了转换：`input[i] = 7 - input[i]`，是出题老师为了增加难度吗：）继续：
   
   ```
   0x000000000040116f <+123>:	mov    $0x0,%esi
   0x0000000000401174 <+128>:	jmp    0x401197 <phase_6+163>
   0x0000000000401176 <+130>:	mov    0x8(%rdx),%rdx
   0x000000000040117a <+134>:	add    $0x1,%eax
   0x000000000040117d <+137>:	cmp    %ecx,%eax
   0x000000000040117f <+139>:	jne    0x401176 <phase_6+130>
   0x0000000000401181 <+141>:	jmp    0x401188 <phase_6+148>
   0x0000000000401183 <+143>:	mov    $0x6032d0,%edx
   0x0000000000401188 <+148>:	mov    %rdx,0x20(%rsp,%rsi,2)
   0x000000000040118d <+153>:	add    $0x4,%rsi
   0x0000000000401191 <+157>:	cmp    $0x18,%rsi
   0x0000000000401195 <+161>:	je     0x4011ab <phase_6+183>
   0x0000000000401197 <+163>:	mov    (%rsp,%rsi,1),%ecx
   0x000000000040119a <+166>:	cmp    $0x1,%ecx
   0x000000000040119d <+169>:	jle    0x401183 <phase_6+143>
   0x000000000040119f <+171>:	mov    $0x1,%eax
   0x00000000004011a4 <+176>:	mov    $0x6032d0,%edx
   0x00000000004011a9 <+181>:	jmp    0x401176 <phase_6+130>
```
上面这部分代码比较难理解，实际包含了两个循环：`<+130>`到`<+139>`以及`<+143>`到`<+169>`。其中`<+163>`到`<+181>`决定了该跳转到哪个循环，只有`input`数组中的值为1时才执行第二个循环。打印出`<+143>`和`<+176>`中的地址0x6032d0，发现它是一个链表。结合这些信息，翻译回C语言，发现这些代码只是根据`input`数组按数序将链表的节点存入另一个数组`nodes`。

`(gdb) x /12xg 0x6032d0`：
```
0x6032d0 <node1>:	0x000000010000014c	0x00000000006032e0
0x6032e0 <node2>:	0x00000002000000a8	0x00000000006032f0
0x6032f0 <node3>:	0x000000030000039c	0x0000000000603300
0x603300 <node4>:	0x00000004000002b3	0x0000000000603310
0x603310 <node5>:	0x00000005000001dd	0x0000000000603320
0x603320 <node6>:	0x00000006000001bb	0x0000000000000000
```

```c
struct node {
    uint64_t value;
    struct node* next;
}* nodes[6];

for (int rsi = 0; rsi != 6; ++rsi) {
    int rcx = input[rsi];
    struct node* rdx = &node1;

    for (int rax = 1; rax != rcx; ++rax) {
        rdx = rdx->next;
    }

    nodes[rsi] = rdx;
}
```

继续看反汇编代码：
```
   0x00000000004011ab <+183>:	mov    0x20(%rsp),%rbx
   0x00000000004011b0 <+188>:	lea    0x28(%rsp),%rax
   0x00000000004011b5 <+193>:	lea    0x50(%rsp),%rsi
   0x00000000004011ba <+198>:	mov    %rbx,%rcx
   0x00000000004011bd <+201>:	mov    (%rax),%rdx
   0x00000000004011c0 <+204>:	mov    %rdx,0x8(%rcx)
   0x00000000004011c4 <+208>:	add    $0x8,%rax
   0x00000000004011c8 <+212>:	cmp    %rsi,%rax
   0x00000000004011cb <+215>:	je     0x4011d2 <phase_6+222>
   0x00000000004011cd <+217>:	mov    %rdx,%rcx
   0x00000000004011d0 <+220>:	jmp    0x4011bd <phase_6+201>
```

以上这段比较好理解，就是根据`nodes`数组按顺序重写了链表各节点的`next`字段，接着看，最后一段了：

```
   0x00000000004011d2 <+222>:	movq   $0x0,0x8(%rdx)
   0x00000000004011da <+230>:	mov    $0x5,%ebp
   0x00000000004011df <+235>:	mov    0x8(%rbx),%rax
   0x00000000004011e3 <+239>:	mov    (%rax),%eax
   0x00000000004011e5 <+241>:	cmp    %eax,(%rbx)
   0x00000000004011e7 <+243>:	jge    0x4011ee <phase_6+250>
   0x00000000004011e9 <+245>:	callq  0x40143a <explode_bomb>
   0x00000000004011ee <+250>:	mov    0x8(%rbx),%rbx
   0x00000000004011f2 <+254>:	sub    $0x1,%ebp
   0x00000000004011f5 <+257>:	jne    0x4011df <phase_6+235>
```

这段也简单，遍历链表，要求链表各节点的低位4字节按从大到小的顺序排列。
综上，最后一关要求输入`1 2 3 4 5 6`6个数字的一个排列顺序，然后将数字`i`转换为`7 - i`，再将预设好的一个链表按顺序重新链接，要求重新链接后的链表各节点的值按从大到小的顺序排列。根据打印出来的链表信息，可以推出答案是`"4 3 2 1 6 5"`。