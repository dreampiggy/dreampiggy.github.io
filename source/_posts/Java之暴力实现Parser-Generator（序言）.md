---
title: Java之暴力实现Parser Generator（序言）
categories: Java
tags:
  - Java
  - 编译原理
updated: '2016-03-30 02:41:13'
date: 2016-03-30 02:40:34
mathjax: true
---

> 编译原理已经学了很多了吧？还有所迷茫？那么今天跟着我一起学习如何暴力写一个Parser Generator

# Why Java：

因为Java有着丰富的对开发人员傻瓜式友好的内置数据结构，什么Map，Set，Stack，求交集求并集也就一句a.contains(b);a.addAll(b)的事情，并且不需要担心资源泄漏(?)的问题，对于我们的暴力实现非常有帮助。而且相比C++我也更为熟悉..

# What is parser generator

这里就指的是支持用户输入CFG（[Context-free grammar][1]），然后生成出一个Java代码，这个代码可以编译以后得到一个Parser用来Parse符合输入CFG定义的文法，类似于[Yacc][2]

迷糊了？举个例子，就是假如用户定义了这样一组CFG，用来匹配一个对于正整数的加法和乘法

$ S \rightarrow TB \\ B \rightarrow TB \mid \epsilon \\ T \rightarrow FT^\* \\ T^\* \rightarrow \* FT^\* \\ F \rightarrow (S) \mid 0 \mid \dots \mid 9 $

当然，数学符号肯定很好写，实际中输入大概是这样子的(\e表示epsilon)：

```text
S  -> T B
B -> + T B | \e
T  -> F T{*}
T{*} -> * F T{*} | \e
F  -> (S) | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

这样的话，这个CFG输给我们的Parser Generator(我们叫做a.java)，运行后得到的代码，再次编译以后(我们叫做b.java)就可以用来匹配输入，对于3 + 5 * 7，我们能给出True的匹配。结果可能是这样子的

```text
LL(1) Parser start

Currernt top: S input: 3
Currernt top: T input: 3
Currernt top: F input: 3
Currernt top: 3 input: 3
Currernt top: T{*} input: +
Currernt top: B input: +
Currernt top: + input: +
Currernt top: T input: 5
Currernt top: F input: 5
Currernt top: 5 input: 5
Currernt top: T{*} input: *
Currernt top: * input: *
Currernt top: F input: 7
Currernt top: 7 input: 7
Currernt top: T{*} input: #
Currernt top: B input: #

Parse result: true
```

当然，这个匹配用的是表驱动的，而且后续会生成AST（[Abstract syntax tree][3]）

# What algorithm

当然，核心在于学习编译，所以我们会分别使用[LR(1)][4]和[LL(1)][5]生成我们的Parser，里面会用到用到LR(1)的三种集合，LL(1)的Closure等知识……

# How about efficiency

看到我说的暴力……你就知道这是妥协了。虽然我们会考虑效率，但只考虑大O，不会在意具体new了多少个对象，是否重复，算法实现也尽量以清晰易懂，直接可以看作伪代码，而不会过多优化，所以，这主要是一个教学，没有过多的实际意义……

# Talk is cheap

自己大概写了个非常简陋的版本……现在只能确保LR(1)文法能够Parse，LL(1)正在努力过测试 GitHub repo: <https://github.com/lizhuoli1126/e-lexer> （不要在意名字……开始想做的是lex却发现走到了类似yacc的路上去了……）

# So what's next

第一篇，我们只大概介绍一下框架，以及一个简单的Input Buffer的实现，还有我们定义的CFG的语法，状态机图，所以大家放轻松，让我们稍后再见

 [1]: https://en.wikipedia.org/wiki/Context-free_grammar
 [2]: https://en.wikipedia.org/wiki/Yacc
 [3]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
 [4]: https://en.wikipedia.org/wiki/LR_parser
 [5]: https://en.wikipedia.org/wiki/LL_parser