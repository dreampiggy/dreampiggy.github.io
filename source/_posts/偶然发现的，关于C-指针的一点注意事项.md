---
title: 偶然发现的，关于C++指针的一点注意事项
categories: C++
tags:
  - C++
updated: '2016-03-30 00:41:01'
date: 2016-03-30 00:38:49
---

C++的指针是大一时期一直觉得头疼的一个东西。当年一直对指针敬而远之。一旦不小心，指针越界的后果就是程序崩溃……> <

然而，偶然一次学妹问到关于动态数组的问题的时候……才偶然发现当年自己学指针似乎一直没有搞清楚这两者的关系：

`pointer`和`pointer[i]`

比如实现一个简单的动态数组，举例就是看如下代码：

第一种方法：

```cpp
#include <iostream>

using namespace std;
int main(int argc, char *argv[]) {
	int arrayInput;
	int arraySize = 0;
	int* arrayPointer;
	cout<<"依次输入数字到这个数组中，输入EOF（Windows下为Ctrl＋Z，Unix/Linux下为Ctrl+D）来停止输入"<<endl;
	while(cin>>arrayInput){
		arrayPointer[arraySize] = arrayInput;
		arraySize++;
	}
	cout<<"现在输出数组中元素"<<endl;
	for(int i = 0;i<arraySize;i++){
		cout<<i<<"元素: "<<arrayPointer[i]<<" ";
	}
}
```

第二种方法：

```cpp
#include <iostream>

using namespace std;
int main(int argc, char *argv[]) {
	int arrayInput;
	int arraySize = 0;
	int* arrayPointer;
	cout<<"依次输入数字到这个数组中，输入EOF（Windows下为Ctrl＋Z，Unix/Linux下为Ctrl+D）来停止输入"<<endl;
	while(cin>>arrayInput){
		*(arrayPointer+arraySize) = arrayInput;
		arraySize++;
	}
	cout<<"现在输出数组中元素"<<endl;
	for(int i = 0;i<arraySize;i++){
		cout<<i<<"元素: "<<*(arrayPointer+i)<<" ";
	}
}
```

直到写完了，才发现自己其实一直没有搞懂，这种arrayPointer[i]表示什么。于是做了一个测试～

```cpp
cout<<"现在输出数组中元素\n第一种："<<endl;
for(int i = 0;i<arraySize;i++){
	cout<<i<<"元素: "<<arrayPointer[i]<<" ";
}
cout<<"\n第二种："<<endl;
for(int i = 0;i<arraySize;i++){
	cout<<i<<"元素: "<<*(arrayPointer + i)<<" ";
}
```

![](http://dreampiggy-image.test.upcdn.net/image/c/b8/7d8680b175f4e74b2de45bd2733e2.png)

试验了一下，才发现，其实arrayPointer[i] 和*(arrayPointer + i) 是等价的，前者是代表从arrayPointer的首地址开始计数，而arrayPointer[0]代表就是arrayPointer指向的地址所对应的值。这一点应该注意一下（其实以前觉得数组和指针挺像的，但是从这一点就知道数组绝对是指针的一小部分子集）。


给自己提个醒吧－ －顺便怀念一下远去的C++。