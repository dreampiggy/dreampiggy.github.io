---
title: Top Down --自顶向下的文法分析
categories: Code
tags:
  - 编译原理
updated: '2016-03-30 02:43:35'
date: 2016-03-29 23:40:13
mathjax: true
---

# Top Down --自顶向下的文法分析

读入：读取下一个待匹配字符进去stack，如果为#且栈顶也为＃，成功；如果此时栈顶不为#，失败
检查：如果栈顶为终结符，回到读入；
否则，随机选择一个规则（假如存在多个规则可以满足的话），使栈顶字符与左部（非终结符）匹配，匹配，用规则的右部（即终结符）替换，整个过程叫做规约(Derivation)
	如果找到匹配的规则，回到读入；
	否则，BackTracking，回溯到上一次的检查，换另一个规则

```swift
if Stack == None Terminal {
	choose rule do divide {
		from right to left, push into stack
	}
}

match {
	pointer move right to read
}


if not match {
	backtrack
}

```


example:

1. $ S \to a \\ S \to Sa $, 给定 "aa"

```text
@1
S# | aa#
a# | aa#
# | a# -> backtrack
S# | aa#
@2
```

存在问题： 

1. 左递归问题：假如一直选择规则2，将导致无穷循环，无法停止     -->解决：设计文法的时候，消除左递归文法，具体见下专题：
$ A \to \*\* \mid \* \\ => \\ A \to \*B \\ B \to \epsilon \\ B \to \* $
2. BackTracking花费时间		-->解决：加入预测，对所有的非终结符加入到预测表中，在进行检查选定一个规则的时候，预读下一个字符，查表检查规则产生的右部是否以预测字符相同，相同继续，不相同直接返回错误，避免过多backtracking，具体见下专题：

## 左递归

1. 定义：
	直接左递归：$ P \to P\alpha \mid \beta \\ $
	间接左递归：$ \\ P \to Aa \\ A \to Bb \\ B \to Pc $，可以存在多个中间非终结符，依次传递
2. 消除：
	直接左递归：$ P \to \beta P^\* \\ P^\* \to \epsilon \\ P^\* \to \alpha P^\* $
	间接左递归：
	1. 首先，对所有参与递归的非终结符，排序，$ B \ A \ P $(其实顺序无关，但是把起始符放在最后，更能简化最终语法，其他顺序产生语法较为复杂）
	2. 依次检查非终结符，如果产生的右部，不含有在该非终结符前面序列的终结符，不处理。比如$ B \to Pc $，$ P $在$ B $之后，故直接处理下一个终结符
		否则，含有在序列之前的终结符，比如$ A \to Bb $含有$ B $，且$ B $在$ A $之前，则直接把所有$ B $使用$ B $的产生式替换，然后处理下一个终结符
	$ A \to Pcb  $
	3. 直到所有的非终结符被处理，其中可能产生直接左递归式：
	$ B \to Pc \\ A \to Pcb \\ P \to Pcba $
	4. 对这个直接左递归式，把$ cba $看作$ \alpha $，使用直接左递归消除：
	$ B \to Pc \\ A \to Pcb \\ P \to P^\* \\ P^\* \to \epsilon \\ P^\* \to cbaP^\* $
	5. 最后一步进行简化，比如此时的$ P \to P^\* $，可替换之后的所有$ P^\* $为$ P $，多行的非终结符合并
	6. 最终结果：$ B \to Pc \\ A \to Pcb \\ P \to cbaP \mid \epsilon $

## 预测表

1. 定义：
	行为所有的终结符以及#，列为所有的非终结符。某一个产生式对应匹配（$ 列 \to 行首字符 $)，比如$ E \to iE^\* $匹配[E,i]。匹配的元素标记为1，不匹配的为0
2. 使用：
	在Top Down的时候，如果检查选择规则时候，查表发现该规则非终结符在表中的标记为1，则继续，否则直接返回错误，无需继续BackTracking


# Bottom Up --自底向上

读入：读取下一个待匹配字符进去stack，如果为#，停机
检查：是否可以被reduction，即可以被某个产生式右部匹配，用左部分替换
如果可以，pop待匹配字符，push那个被reduct的字符进去stack，回到检查
如果不行，回到读入

存在问题：
1. 对于重复的字符匹配，比如 $ A \to \*\* $，出现一个＊就会被规约，导致错误。使用Handling来特殊处理$ \epsilon $