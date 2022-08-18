---
layout: post
title: "面试日志 - 2"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - note
---

华为2012实验室 GaussDB 实习面试总结：

实习招聘我和室友都是两轮面试，一轮技术面和一轮主管面。我机试过了第一题和第二题90%用例。第一题概括来说是有一个随机生成数组（无重复），求数组每个元素右边比它小的元素个数，不过数据范围比较大，是[1, 100000]，直接暴力 $O(n^2)$ 的话会超时，我是从右往左遍历，然后维护一个有序数组，每访问一个新元素就二分找它在有序数组里的位置，它前面的就是所有比它小的元素，然后再把它插进数组里，虽然理论上数组的 insert 应该是线性复杂度，但是反正这题过了;第二题是一个围棋棋盘问黑子白子各自的活棋个数是多少，dfs 就行，但是给的用例有问题，说好输入的第一个数给出棋盘的长宽（矩阵是方的）然后接着是表示棋子的字符，但是第二个用例给出 5×5 矩阵，文字解释用例也说是5×5，后面给出的输入却有30个，然后它输出的结果也是按照 5×6 的棋盘来计算的，就尼玛离谱。我当然可以根据字符的长度和给出的行数再计算棋盘的大小，但是这是它题目的问题，为什么要我背锅。反正拿了90%分数了，我直接交卷溜了，第三题看都不看。

心理测评听说给我挂了，HR 让我重做一次。好像我周围投华为的人里，心理测评一轮没过的人还不少。说实话我有点怀疑这个测评的可靠程度的，只靠一份问卷真的能调查出个人的心理状况吗？

技术面，面试官问的问题多少有点奇怪。上来问我，什么是分布式事务，为什么需要分布式事务。阿西，还真没见过有人这样问问题的。还问了项目里 Percolator 两阶段提交有什么不同，我尝试给他解释，但是他没耐心听，然后问 Raft 如何保证数据一致，我只讲了 Raft 的 Strong Leader 他就没往下接着问，。大概二十分钟就给我现场出算法题，写了个 `select sum(c2) from t group by c1` 的 SQL，然后给我一个函数签名让我实现，只跟我说这个函数就是实现这个 SQL 语句的 group by，“就这么简单”。剩下的信息全踏马是我一个个追问出来的，他原本好像真的就不打算再继续说明了。

```
// 最开始给我的函数签名，可以看看一共有多少问题
long GroupByOperator(char *col1[], char *col2[], int row_num);
```

首先，在他没给我创建表的语句的情况下，根据这个函数的入参，我们看到表示两列数据的类型都是 char **，对字符串求和会有什么表现？我提出这个问题后，面试官让我把第二个入参改为 int 类型；然后是这个返回值，他给我一个 long 类型。但是 group by 之后求和，应该得到是每个 group 里 col 2 列数据的和，应该是一个数组，我再提出这个问题之后，他让我把返回值改成 long 数组。然后我问他可以把用 c++ 写吗，得到允许之后我就把函数改成了这样

```
vector<long> GroupByOperator(vector<char> col1, vector<int> col2);
```

他居然问我返回数组的长度呢，啧，说实话有点不想搭理他了。一开始用例不给，数据类型不给，数据范围不给，自己觉得问题很简单但是题目都能给错，我问他给多久实现，他反问我”你想要多久”。最后实现也就是用一个哈希表来做。然后他问我，如果不用 STL 的 unordered_map，而是给一个假定性能很好的散列函数，怎么做这个哈希映射，写一下代码，还说这是想考察你的系统设计能力。阿一西，累了赶紧毁灭吧，真就一点信息不给全得自己假设啊？数据范围呢？就算我选一个最简单的拉链解决冲突，那性能用考虑吗？还有那么多种解决哈希冲突的方法，你是要我一个个写给你看还是咋地？我理解上程序语言里提供的哈希表的实现都是不断改进的，我们当然可以随便选几种简单的策略就做一个哈希表，但是性能必然没法跟标准库中的相提并论。要做一个比较好的实现，肯定得结合场景（在这里是具体的数据量）来进行，不然都踏马是在扯淡，没有意义。

这个问题结束之后，面试官还问了几个 C++ 虚函数相关的问题。但是总的来说我感觉他一直在改自己的问题，好几次都是刚说完问题，我还没开始回答，他又换了一个新的问题，感觉对自己想问的东西都没有比较清晰的思路。最后也没留提问的时间，我问他下一轮面试时间，他说这个你问 HR，就挂断了，好像连面试结束的客套话都不愿说一句。