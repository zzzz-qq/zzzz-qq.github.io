---
title: 一个C++14/SDL2 实现的比较完善的俄罗斯方块
date: 2018-11-15 00:17:14
tags:
	- C++
	- SDL2
	- Tetris
---
&emsp;&emsp;最近工作遇到一个问题：要在几个继承自同一基类的子类初始化的时候执行一些特定的初始化操作，想当然地，在基类写个空的虚函数，子类override，然后在基类的构造函数中调用这个虚函数。一执行，oh，才发现子类override的虚函数都没被调用...<!-- more -->想了一秒钟，反应过来：构造函数中的虚函数是不会被动态绑定的，换句话说，在基类构造函数中调用的虚函数只能是基类自己定义的虚函数，而不是子类override的那个。我能清楚地记得《Effecitive C++》一书中有个item讲过这个问题，我还看过不止一遍，然而一不小心还是会踩坑。C++的坑太多，平时工作用的最多的都是C with class，type traits，久而久之，踩过的坑就都忘了。C++需要不断练习，这次先用SDL2写个俄罗斯方块练练手，代码及说明在 [GitHub: Tetirs](https://github.com/zzzz-qq/toys/tree/master/tetris)。

![tetris](/tetris/tetris.png)