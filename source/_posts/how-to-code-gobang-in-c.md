---
title: 如何用C语言写一个五子棋
date: 2017-01-20 16:53:59
tags:
- C
- Game
categories:
- C
- Game
---

为什么突然想起来写这么个东西，大概是有一天，突然想测试下自己的C语言到底还记得多少。
就找了年末没什么事情的时候，花了大概2个小时撸了一个这么个东西。
[renju-c 源码下载](https://github.com/yannxia-self/renju-c)

<!-- more -->

## 我们需要怎么个东西
大概是我们需要一个黑框框里面能过下五子棋的东西。

## 完善需求
我们需要的是 能够双人进行五子棋游戏的一个小程序。

## 我的计划是
我需要这么几个函数

- 显示棋盘
- 下棋
- 判断胜负

## 如何实现我的计划
简单的画了一个图
<iframe id="embed_dom" name="embed_dom" frameborder="0" style="border:1px solid #000;display:block;width:430px; height:400px;" src="https://www.processon.com/embed/5881d2bbe4b098bf4cee4a2a"></iframe>

### 1. 绘图的逻辑
标准的五子棋是 19 x 19，那我们很直觉的使用一个二维数组.初始化都是 *，绘图直接打印出这个棋盘就好了。

```c
#define CHESSBOARD_SIZE 19 //19 * 19 size
char chessboard[CHESSBOARD_SIZE][CHESSBOARD_SIZE]; //整个棋盘

void init()
{
    int i, j;
    for (i = 0; i < CHESSBOARD_SIZE; i++)
    {
        for (j = 0; j < CHESSBOARD_SIZE; j++)
        {
            chessboard[i][j] = '*';
        }
    }
}
```


### 2. 判断的逻辑
这个项目最复杂的就是判断如何成立了，我的逻辑是这样的。

蓝色的在中间，就是我们需要判断的棋子，我们可以从 竖横的方向 和 斜的方向去判断是否有五个字。

比如图示中，我们假设 蓝色点是 (a,b) 点，我们先从左侧找，最长的同色我们发现是2个，我们再向后找最长同色，我们发现是0个，那我们只有 2+1[自身] = 3 就是3个。

<iframe id="embed_dom" name="embed_dom" frameborder="0" style="border:1px solid #000;display:block;width:800px; height:400px;" src="https://www.processon.com/embed/5881d5bae4b049e795be4834"></iframe>

具体的参见代码

### 3. 下棋的逻辑
这个逻辑就不是很复杂，就是分为检测，不允许下在同一个位置，并且在棋盘内，就是一个Scanf的问题。


## 一些细枝末节
### 1. 关于清屏幕
因为清屏幕是必须的··保证游戏的可玩性，一开始我调用了 system("clear") ,不过这里的问题有了，如果是在windows上，这个时候，我们唯一能做的就是是否能够在运行时判断操作性，不过好像做不到，
只能找到一个基于编译器的判断编译器的方式

```c
void clearscr(void)
{
#ifdef _WIN32
    system("cls");
#elif defined(unix) || defined(__unix__) || defined(__unix) || (defined(__APPLE__) && defined(__MACH__))
    system("clear");
//add some other OSes here if needed
#else
#error "OS not supported."
//you can also throw an exception indicating the function can't be used
#endif
}
```

### 2. 如何处理 0，0 点
这个时候遇见的问题就是 左边偏移量是越界的怎么办，两个办法。
- 把棋盘变成 30 * 30 ,我们只有中间 19*19
- 处理越界下标都处理为 棋盘属性

最后选择了第二种

```c
char get_piece(int x, int y)
{
    if (BETWEEN_IN_SIZE(x + 1) && BETWEEN_IN_SIZE(y + 1))
    {
        return chessboard[x][y];
    }

    return '*';
}
```

## 总结
算了完成了自己多年之前想做了一个东西，实际上的确不是很复杂。给C初学者了一点小小的启示。如果有问题可以在下面提或者在Github，我会尽量回答的。

## Next
准备下次做一个自动下棋的AI，大家期待一下吧。撒花完结。
