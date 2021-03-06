---
layout: post
title: "栈内存越界血案引发的思考"
date: 2017-10-14
excerpt: "各种Memory"
tags: [stack, 内存, C++]
comments: true
---

## 什么是栈内存
C++程序一共有五种内存（或者说内存分为五个区）
1. 栈内存——stack，系统分配和回收，win下一般大小为2M，操作方式同数据结构的栈
2. 堆内存——heap，用户申请和释放，不同于数据结构中的堆，操作方式倒是与链表相似
3. 全局区（静态区）——static，用于存放全局变量和静态变量，初始化的全局变量和静态变量放在一起，未初始化的全局变量和静态变量放在一起，程序结束时被释放
4. 文字常量区——用于存放常量字符串，程序结束后被释放
5. 程序代码区——存放函数的二进制代码


```
//main.cpp
inta = 0; //全局初始化区
char*p1; //全局未初始化区
main()
{
int b; //栈 
   char s[] = "abc"; //栈 
   char *p2; //栈  
   char *p3 = "123456"; //123456\0在常量区，p3在栈上。  
   static int c =0； //全局（静态）初始化区 
   p1 = (char *)malloc(10);  
    p2 = (char*)malloc(20);  
   //分配得来的10和20字节的区域就在堆区, 但是注意p1、p2本身是在栈中的 
   strcpy(p1,"123456"); //123456\0放在常量区，编译器可能会将它与p3所指向的"123456"优化成一个地方。  
} 
```


虚拟内存的分配图：
![image](http://img.my.csdn.net/uploads/201608/08/1470644707_7478.png)

可以看出栈是由高地址向低地址生长，而堆是低地址向高地址生长的。所以栈的分配是有大小限制的，栈由系统自动分配，在win下一般是2m大小，是一块连续的内存区域。而堆内存是由用户主动申请释放的，速度比较慢，并不连续，大小受限于计算机虚拟内存，所以可用空间一般很大。

### 栈和堆存储内容的区别
栈：
由系统自动分配一块连续的内存空间，将调用函数指令的下一条指令压入栈中，然后将函数参数依次压入栈中（一般编译器是从右到左），然后再将函数体中的局部变量压入栈中，函数结束的时候将栈中的内容依次出栈，根据栈底的下一条指令的地址找到并执行下一条指令，程序继续运行。

堆：
由用户申请并释放的非连续的内存空间，堆由底地址向高地址生长，第一个字节存放堆的长度，堆的内容由用户决定。

## 栈数组越界引发的血案

由于栈是顺序向下生长的，存的内容依次是：
- 下一条指令的地址
- 参数（一般编译器是从右到左）
- 局部变量

所以如果某个局部变量后越界，就会踩到上个变量的内容，甚至是函数参数或者函数返回地址（函数下一条指令的地址）导致严重的线程栈崩溃。如果数组越界不是很大，还在线程栈的范围内，不会造成违规的内存操作，所以编译器会给予放行（DEBUG模式下如果运行过程中发生踩内存VS会报错）。
还有一种可能导致线程栈被破坏的操作就是在函数中返回局部变量的地址，如果该地址正好被某个参数所用，在函数外部修改了这个地址的内容，就会造成线程栈的破坏和程序的崩溃。
所以一旦有程序的线程栈被破坏，可以看看是否有局部变量的地址被返回或者数组越界了。

由于栈内存的特点，栈内数组越界还有可能造成死循环

```
void func(...){
    int i;
    int a[5];
    for(i = 0; i < 6; i++){
        a[i] = 0;
    }
}
```
上面这段程序会陷入死循环，这是为什么呢？因为a[6]踩到了i，导致在第六次迭代的时候，地址&a[6]中的内容被赋值为0，也就是i被赋值为0，导致循环继续。

换种写法：
```
void func(...){
    int a[5];
    int i;
    for(i = 0; i < 6; i++){
        a[i] = 0;
    }
}
```
看似不会发生死循环，但这个时候a[6]已经踩到左边第一个参数，问题貌似更严重哦，所以写代码什么的三观得正，不要乱出轨，否则编译器都救不了你。
