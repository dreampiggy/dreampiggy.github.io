---
title: Retroactive Data Structure
categories: Code
tags:
  - 数据结构
updated: '2016-03-30 02:34:16'
date: 2016-03-30 02:30:31
mathjax: true
---

> MIT Open Course

[MIT-retroactive-data-structure][1]

# Time line

Insert(time, ops); Delete(time, ops); Query(time, query);

How to insert ops into two times? -> BST || linked list

How to deal with side effect? -> save all the ops except `delete`

`Partial Retroactive`

Query always done at present => time = $ \infty $

`Full Retroactive`

Query can done at whatever time

# What about formal defination of partial & full

$ query(t,A \cup B) \equiv f(query(t,A),query(t,B)) , f \ is \ O(1)$

# Implementation

`segment tree` : BST on time

![title][2]

`Commutative updates` => 可交换updates

$ x.y = y.x $

`Invertible updates` => 可逆updates

$ x.x^{-1} = \emptyset $

Delete(time, ops) === Insert(now, ops^{-1})

# Practice

Dynamic hashing

Priority queue

*   query will cost $ O(lgn) $
*   store in BBST (balanced binary search tree)
*   leaves : insert => store by time not value

General Transformation

*   rollback: change r time in the past will cost $ O(r) $

*   lower bound: $ \Omega(r) $

# More complicated - Non-oblivious Retroactive

which means after one insert or delete ,you can get `the query that change`

# Just a little bit code

`retroactive queue`: [GitHub-Functional.js][3]


Reference: [Retroactive Data Structures - ERIK D. DEMAINE, JOHN IACONO, STEFAN LANGERMAN][4]

 [1]: http://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-851-advanced-data-structures-spring-2012/lecture-videos/session-2-retroactive-data-structures/
 [2]: https://upload.wikimedia.org/wikipedia/commons/e/e5/Segment_tree_instance.gif
 [3]: https://github.com/lizhuoli1126/Functional.js#retroactivedata-structure
 [4]: http://erikdemaine.org/papers/Retroactive\_TALG/paper.pdf