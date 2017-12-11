---
layout: post
title: "（Access violation）内存非法访问问题查找实战"
date: 2017-12-11
excerpt: "掌握一点DEBUG技术，别说12月了，全世界都会对你好一点"
tags: [内存非法访问, 空指针, C++]
comments: true
---

## 前沿
作为一名C++ boy，“Access violation”这样的异常实在是司空见惯，C++赤裸的内存管理，在编写代码的时候数组越界，空指针等等问题都会让你的程序抛出“Access violation”。在Debug阶段你可以很快的定位问题，因为一有“Access violation”，机智的IDE会立刻定位到非法访问内存的代码行，但是线上呢？之前面对这样的问题我也是一脸懵逼，后来学习了一些‘高级’的DEBUG手段后事情就看起来没有原来那么难搞，甚至可以说是轻而易举，下面我就来讲讲今天在线上发现的一个空指针访问的异常（崩了一个小时吧，眼泪都快出来了）。

## 准备条件
- 线上崩溃服务器的dump文件（你可以理解为就是当时服务器的内存镜像，full_dump甚至还有堆数据）
- 编译线上服务器生成的pdb文件（程序调试数据库，微软是这么说的，里面有一些结构体，函数相关的信息，就是一个符号表吧，用来定位错误发生地和还原数据的）
- 源码（当然是review啦，机智的windbg甚至可以直接给你指出案发地点）

## 开始分析
1. 把dump拖进windbg中
2. 加载线上服务器的pdb文件，微软的pdb和服务器源码（可以不用）
3. 输入命令.reload加载符号文件
> 如果你是minidump，windbg会亲切的提示你，只有寄存器，栈和部分内存信息可用（已经足够你查到问题线程和栈了，但是你要想知道出问题时候的堆数据，没门）

```
windbg:
User Mini Dump File: Only registers, stack and portions of memory are available
```
> 如果你是fulldump，windbg会提示你（下面我们采用fulldump分析，因为我要看看堆数据到底是怎样的，而不仅仅是查到问题代码）

```
windbg:
User Mini Dump File with Full Memory: Only application data is available
```
4. 什么都不说，先来波智能分析，命令!analyze -v

```
windbg:
...
FAULTING_IP: 
***!CAssitServer::OnCardsActivityInfo+47 [e:****\mainserver.cpp @ 1877]
00f35de7 8b02            mov     eax,dword ptr [edx]

EXCEPTION_RECORD:  ffffffff -- (.exr 0xffffffffffffffff)
ExceptionAddress: 00f35de7 (GssuAssitSvrcd!CAssitServer::OnCardsActivityInfo+0x00000047)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 00000000
   Parameter[1]: 00000000
Attempt to read from address 00000000

PROCESS_NAME:  ****.exe

ADDITIONAL_DEBUG_TEXT:  
Use '!findthebuild' command to search for the target build information.
If the build information is available, run '!findthebuild -s ; .reload' to set symbol path and load symbols.

FAULTING_MODULE: 76ec0000 ntdll

DEBUG_FLR_IMAGE_TIMESTAMP:  5a2e6780

ERROR_CODE: (NTSTATUS) 0xc0000005 - 0x%p

EXCEPTION_CODE: (NTSTATUS) 0xc0000005 - 0x%p

EXCEPTION_PARAMETER1:  00000000

EXCEPTION_PARAMETER2:  00000000

READ_ADDRESS:  00000000 

FOLLOWUP_IP: 
****!CAssitServer::OnCardsActivityInfo+47 [*****\mainserver.cpp @ 1877]
00f35de7 8b02            mov     eax,dword ptr [edx]

FAULTING_THREAD:  00004120

BUGCHECK_STR:  APPLICATION_FAULT_NULL_POINTER_READ_WRONG_SYMBOLS

PRIMARY_PROBLEM_CLASS:  NULL_POINTER_READ

DEFAULT_BUCKET_ID:  NULL_POINTER_READ

LAST_CONTROL_TRANSFER:  from 00f351b9 to 00f35de7

STACK_TEXT:  
05c5f59c 00f351b9 01638048 0169e3b8 01679550 GssuAssitSvrcd!CAssitServer::OnCardsActivityInfo+0x47 [****\mainserver.cpp @ 1877]
05c5f5f4 00f6b130 01638048 0169e3b8 5d18635c ****!CAssitServer::OnRequest+0x3f9 [*****\mainserver.cpp @ 1167]
05c5f760 00f4f441 05c5f92c 05c5f854 01679870 GssuAssitSvrcd!CIocpWorker::DoWorkLoop+0x250
05c5f84c 00f4f353 5ee437b0 5ee437b0 01679870 GssuAssitSvrcd!CBaseWorker::WorkerThreadProc+0xd1
05c5f92c 5ee43651 011eee34 35ed94f3 5ee437b0 GssuAssitSvrcd!CBaseWorker::WorkerThreadFunc+0x33
WARNING: Stack unwind information not available. Following frames may be wrong.
05c5f968 5ee43861 01664618 05c5f988 73f362c4 MSVCR120D!beginthreadex+0x1a1
05c5f974 73f362c4 01664618 73f362a0 32d3dadf MSVCR120D!endthreadex+0x181
05c5f988 76f20f79 01679870 a19c4b46 00000000 kernel32!BaseThreadInitThunk+0x24
05c5f9d0 76f20f44 ffffffff 76f42eb9 00000000 ntdll!RtlSubscribeWnfStateChangeNotification+0x439
05c5f9e0 00000000 5ee437b0 01679870 00000000 ntdll!RtlSubscribeWnfStateChangeNotification+0x404
...
```
哇，是不是马上就定位到代码行了，并且还亲切的告诉你的异常类型（Access violation）和代码文件，甚至还贴心的帮你切换到了出问题的线程并恢复当时上下文输出栈信息（windbg是解决了多少程序员掉发问题）。
5. 分析对应的源码的代码行，OnCardsActivityInfo函数的第二个参数的一个成员变量（指针）空了
6. 为什么会空呢？（我想看看第二个参数当时的数据）
7. 输入命令dd 0169e3b8（这是第二个参数的地址）

```
windbg:
0:034> dd 0169e3b8
0169e3b8  00000000 00061e6d 00000000 00000000
0169e3c8  00000000 00000033 00000000 00000000
0169e3d8  fdfdfdfd 00720044 27324a90 80000265
0169e3e8  003a0043 0057005c 004e0049 004f0044
0169e3f8  00530057 0053005c 00530059 00450054
0169e408  0033004d 005c0032 00490057 004d004e
0169e418  0042004d 00530041 002e0045 006c0064
0169e428  0000006c 0000006c 27084a6a 80000300
```
> 满屏Hex看不懂啊喂，能不能弄点人懂的数据，可以！

8. view->watch操作如下:

![观察某个地址的数据（强转成某种结构体）](assets/post/accessviolation/1.png)

9. 或者你想像在vs中调试一样看到locals变量
    - view->call stack
    - view->locals
    - 输入命令~34s; .ecxr; kb（34是刚才出问题的线程，.ecxr是恢复上线文，kb是输出栈）

> 然后你就会看到熟悉的场面如下

![image](assets/post/accessviolation/2.png)

## 总结
这次崩溃还好只是在一个协助服务器上发生的，自动管理工具不断的在“复活”它，几十分钟内生成了好多dump文件，原因是这个服务器在shutdown的时候会重复释放内存（其实也是NULL指针导致的Access Violation），但是我并没有管它，嘻嘻，很皮。就像DEBUG阶段排除错误一样，当有上千个错误发生的时候一定要先找第一个错误（永远记得大一时候老师的这句话），几十个dump文件也应该从第一个找起，实际复查的时候，后面的几十个dump都是一个错误导致的。除了重构，DEBUG应该是程序员迭代能力最好的一个渠道了吧

糟糕代码-》分析问题-》解决问题-》Level Up
