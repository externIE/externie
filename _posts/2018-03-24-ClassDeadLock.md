---
layout: post
title: "175个线程中找到两个互相等待的线程（经典死锁问题调查）"
date: 2018-03-19
excerpt: "经典死锁问题调查"
tags: [死锁, windbg, debug]
comments: true
---

# 背景
晚上正在吃鸡的时候，一通电话：“糟了，服务器无响应了”，“。。。。额。。。我远程看看”。cpu，内存正常，无dump生成，日志输出停止，服务器确实“假死了”。初步判断应该是工作线程在等待某个临界区。让运维手动转储服务器dump文件，拿到public环境下编译的pdb文件和exe文件，然后再让运维重启服务器。开始调查。。。。

# 背景知识
## pbd文件
程序数据库文件，存储调试win程序的一些基本信息，函数名，行号，文件名，栈帧等等。每个模块的每次编译都会生成一个与之匹配的pdb，即便你根本没有动你的源码。

[更多关于PDB文件](https://www.cnblogs.com/feihe0755/p/6261938.html)

## dump文件
dump文件即转储文件，是程序的内存镜像，mini dump包含栈，寄存器和部分内存信息，而full dump还包含堆信息。你可以用任务管理器生成相应进程的转储文件，也可以在程序发生崩溃（发生未捕获异常的时候）自动生成转储文件。

[更多关于dump文件](https://blog.csdn.net/GG_SiMiDa/article/details/72725385)

> dump文件你可以认为是二进制的案发现场，但它是不可读的，通过pdb文件你可以真正将案发现场转为人类可读的方式，就像你在vs上调试代码一样方便。

# 开始调查
### 0. 准备
在windbg中加载dump，程序的pdb和windows的pdb。

由于没有产生崩溃，也就是程序其实并没有抛出异常，我们就不用万能的!analyze -v命令来分析程序了。因为初步判断是死锁问题，所以只要看看程序的线程栈是怎样的即可。一个注意点是，我们的程序是win32平台上编译的32位程序，所以dump应该与之匹配也是32位才行，但是如果运维用的64位的任务管理器截取的dump就会是64位的，这种情况下我们首先得对dump文件进行64位到32位的转换。命令如下：

```
.load wow64exts; !sw
```
下面是我的调查步骤：
###  1. 我要看看所有线程栈的情况
输入命令：
```
~*kv
```
输出：

```
（175个线程的栈解退情况）
```
### 2. 根据栈解退情况找到8个客户端消息处理线程
51号到58号线程


```
  51  Id: 146c.b54 Suspend: 1 Teb: 7ef09000 Unfrozen
ChildEBP RetAddr  Args to Child              
079af3c4 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
079af428 776fea92 00000000 00000000 008a27e0 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
079af450 001c5279 008a2bd4 75bcfd54 008a27d8 ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
079af480 001913f0 00004101 079af4b0 75bcf1a0 gssssvr!CBaseServer::GetRoomPtr+0x39
079af874 0018e424 14839818 141e5940 00893f40 gssssvr!CMainServer::OnTalkToTable+0xf0
079af890 001f46ae 14839818 141e5940 008a27d8 gssssvr!CMainServer::OnRequest+0x3c4
079af8a8 00138320 14839818 141e5940 75bcf10c gssssvr!CSkServer::OnRequest+0xae
079af8d4 002055c4 14839818 141e5940 008a27e0 gssssvr!CGameServer::OnRequest+0x120 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 197]
079af8fc 001f92bb 00000000 0472acb8 0472a8f0 gssssvr!CIocpWorker::DoWorkLoop+0xa4
079af914 001f928b 079af954 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
079af91c 7194c01d 008a27d8 da5dfc17 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
079af954 7194c001 00000000 079af96c 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
079af960 7623336a 0472a8f0 079af9ac 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
079af96c 776e9902 0472a8f0 df5e8300 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
079af9ac 776e98d5 7194bfb4 0472a8f0 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
079af9c4 00000000 7194bfb4 0472a8f0 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  52  Id: 146c.23a0 Suspend: 1 Teb: 7ef06000 Unfrozen
ChildEBP RetAddr  Args to Child              
07c9ebd0 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
07c9ec34 776fea92 00000000 00000000 008a27d8 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
07c9ec5c 001c5279 008a2bd4 75efe558 13ae36e0 ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
07c9ec8c 00196118 00004338 07c9ecb0 75eff84c gssssvr!CBaseServer::GetRoomPtr+0x39
07c9f198 00188131 00004338 0000000e 00000004 gssssvr!CMainServer::YQW_CheckOnline+0x58
07c9f6ac 0018e168 13d46b60 0f928de0 00893f90 gssssvr!CMainServer::OnGamePulse+0x141
07c9f6c8 001f46ae 13d46b60 0f928de0 008a27d8 gssssvr!CMainServer::OnRequest+0x108
07c9f6e0 00138320 13d46b60 0f928de0 75effec4 gssssvr!CSkServer::OnRequest+0xae
07c9f70c 002055c4 13d46b60 0f928de0 008a27e0 gssssvr!CGameServer::OnRequest+0x120 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 197]
07c9f734 001f92bb 00000000 0090a058 0472acb8 gssssvr!CIocpWorker::DoWorkLoop+0xa4
07c9f74c 001f928b 07c9f78c 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
07c9f754 7194c01d 008a27d8 da0ef2cf 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
07c9f78c 7194c001 00000000 07c9f7a4 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
07c9f798 7623336a 0472acb8 07c9f7e4 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
07c9f7a4 776e9902 0472acb8 df0d8d48 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
07c9f7e4 776e98d5 7194bfb4 0472acb8 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
07c9f7fc 00000000 7194bfb4 0472acb8 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  53  Id: 146c.c68 Suspend: 1 Teb: 7ef03000 Unfrozen
ChildEBP RetAddr  Args to Child              
07e4f764 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
07e4f7c8 776fea92 00000000 00000000 008a27e0 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
07e4f7f0 000e2191 008a2bd4 008a2bd4 07e4f80c ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
07e4f800 000e18e9 07e4f830 07e4f854 00159484 gssssvr!CCritSec::Lock+0x11 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\libraryvc12\uwl9.1\uwlutil.h @ 137]
07e4f80c 00159484 008a2bd4 75c2f180 00000000 gssssvr!CAutoLock::CAutoLock+0x19 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\libraryvc12\uwl9.1\uwlutil.h @ 160]
07e4f854 0013fad9 149f98d0 0f8cbe78 75c2f4d8 gssssvr!CGameServer::TransmitEnterGame+0x64 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 10578]
07e4fd0c 0018e0ad 149f98d0 14d3dae0 00893f80 gssssvr!CGameServer::OnEnterGame+0x1bb9 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 6207]
07e4fd28 001f46ae 149f98d0 14d3dae0 008a27d8 gssssvr!CMainServer::OnRequest+0x4d
07e4fd40 00138320 149f98d0 14d3dae0 75c2f4a4 gssssvr!CSkServer::OnRequest+0xae
07e4fd6c 002055c4 149f98d0 14d3dae0 008a27e0 gssssvr!CGameServer::OnRequest+0x120 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 197]
07e4fd94 001f92bb 00000000 0472a8f0 0472b080 gssssvr!CIocpWorker::DoWorkLoop+0xa4
07e4fdac 001f928b 07e4fdec 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
07e4fdb4 7194c01d 008a27d8 da23f8af 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
07e4fdec 7194c001 00000000 07e4fe04 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
07e4fdf8 7623336a 0472b080 07e4fe44 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
07e4fe04 776e9902 0472b080 df2084e8 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
07e4fe44 776e98d5 7194bfb4 0472b080 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
07e4fe5c 00000000 7194bfb4 0472b080 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  54  Id: 146c.e98 Suspend: 1 Teb: 7ef00000 Unfrozen
ChildEBP RetAddr  Args to Child              
07b9f7f8 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
07b9f85c 776fea92 00000000 00000000 008a27e0 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
07b9f884 001899f0 008a2bd4 759ff068 14202408 ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
07b9f9bc 0018e217 13e1a4f8 14202408 00893f70 gssssvr!CMainServer::OnGetTableInfo+0x190
07b9f9d8 001f46ae 13e1a4f8 14202408 008a27d8 gssssvr!CMainServer::OnRequest+0x1b7
07b9f9f0 00138320 13e1a4f8 14202408 759ff3f4 gssssvr!CSkServer::OnRequest+0xae
07b9fa1c 002055c4 13e1a4f8 14202408 008a27e0 gssssvr!CGameServer::OnRequest+0x120 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 197]
07b9fa44 001f92bb 00000000 0472b080 0475f698 gssssvr!CIocpWorker::DoWorkLoop+0xa4
07b9fa5c 001f928b 07b9fa9c 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
07b9fa64 7194c01d 008a27d8 da7effdf 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
07b9fa9c 7194c001 00000000 07b9fab4 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
07b9faa8 7623336a 0475f698 07b9faf4 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
07b9fab4 776e9902 0475f698 df7d8058 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
07b9faf4 776e98d5 7194bfb4 0475f698 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
07b9fb0c 00000000 7194bfb4 0475f698 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  55  Id: 146c.1f24 Suspend: 1 Teb: 7eefd000 Unfrozen
ChildEBP RetAddr  Args to Child              
0803e2c4 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
0803e328 776fea92 00000000 00000000 008a27e0 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
0803e350 000e2191 008a2bd4 008a2bd4 0803e36c ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
0803e360 000e18e9 0803e4c0 0803fadc 00139418 gssssvr!CCritSec::Lock+0x11 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\libraryvc12\uwl9.1\uwlutil.h @ 137]
0803e36c 00139418 008a2bd4 7a25f308 008a27d8 gssssvr!CAutoLock::CAutoLock+0x19 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\libraryvc12\uwl9.1\uwlutil.h @ 160]
0803fadc 001382a0 13ec17c0 0f9dad58 00893f60 gssssvr!CGameServer::OnSendSysMsgToServer+0x1a8 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 563]
0803fb0c 002055c4 13ec17c0 0f9dad58 008a27e0 gssssvr!CGameServer::OnRequest+0xa0 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 164]
0803fb34 001f92bb 00000000 0475f698 0475fa60 gssssvr!CIocpWorker::DoWorkLoop+0xa4
0803fb4c 001f928b 0803fb8c 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
0803fb54 7194c01d 008a27d8 d5c4fecf 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
0803fb8c 7194c001 00000000 0803fba4 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
0803fb98 7623336a 0475fa60 0803fbe4 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
0803fba4 776e9902 0475fa60 d0c78148 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
0803fbe4 776e98d5 7194bfb4 0475fa60 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
0803fbfc 00000000 7194bfb4 0475fa60 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  56  Id: 146c.1b0c Suspend: 1 Teb: 7eefa000 Unfrozen
ChildEBP RetAddr  Args to Child              
0826e054 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
0826e0b8 776fea92 00000000 00000000 008a27e0 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
0826e0e0 000e2191 008a2bd4 008a2bd4 0826e0fc ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
0826e0f0 000e18e9 0826e250 0826f86c 00139418 gssssvr!CCritSec::Lock+0x11 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\libraryvc12\uwl9.1\uwlutil.h @ 137]
0826e0fc 00139418 008a2bd4 7a00f1b8 008a27d8 gssssvr!CAutoLock::CAutoLock+0x19 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\libraryvc12\uwl9.1\uwlutil.h @ 160]
0826f86c 001382a0 1487b6f0 14c7c958 00893f50 gssssvr!CGameServer::OnSendSysMsgToServer+0x1a8 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 563]
0826f89c 002055c4 1487b6f0 14c7c958 008a27e0 gssssvr!CGameServer::OnRequest+0xa0 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 164]
0826f8c4 001f92bb 00000000 0475fa60 0475fe28 gssssvr!CIocpWorker::DoWorkLoop+0xa4
0826f8dc 001f928b 0826f91c 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
0826f8e4 7194c01d 008a27d8 d5e1fc5f 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
0826f91c 7194c001 00000000 0826f934 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
0826f928 7623336a 0475fe28 0826f974 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
0826f934 776e9902 0475fe28 d0e283d8 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
0826f974 776e98d5 7194bfb4 0475fe28 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
0826f98c 00000000 7194bfb4 0475fe28 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  57  Id: 146c.1644 Suspend: 1 Teb: 7eef7000 Unfrozen
ChildEBP RetAddr  Args to Child              
0841ed80 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
0841ede4 776fea92 00000000 00000000 008a27d8 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
0841ee0c 001c5279 008a2bd4 7a67e7e8 008be488 ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
0841ee3c 00196118 00004101 0841ee60 7a67fa9c gssssvr!CBaseServer::GetRoomPtr+0x39
0841f348 00188131 00004101 00000020 00000002 gssssvr!CMainServer::YQW_CheckOnline+0x58
0841f85c 0018e168 14839798 14fb6fa8 00893f30 gssssvr!CMainServer::OnGamePulse+0x141
0841f878 001f46ae 14839798 14fb6fa8 008a27d8 gssssvr!CMainServer::OnRequest+0x108
0841f890 00138320 14839798 14fb6fa8 7a67f114 gssssvr!CSkServer::OnRequest+0xae
0841f8bc 002055c4 14839798 14fb6fa8 008a27e0 gssssvr!CGameServer::OnRequest+0x120 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 197]
0841f8e4 001f92bb 00000000 0475fe28 047601f0 gssssvr!CIocpWorker::DoWorkLoop+0xa4
0841f8fc 001f928b 0841f93c 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
0841f904 7194c01d 008a27d8 d586fc7f 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
0841f93c 7194c001 00000000 0841f954 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
0841f948 7623336a 047601f0 0841f994 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
0841f954 776e9902 047601f0 d0858338 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
0841f994 776e98d5 7194bfb4 047601f0 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
0841f9ac 00000000 7194bfb4 047601f0 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

  58  Id: 146c.d70 Suspend: 1 Teb: 7eef4000 Unfrozen
ChildEBP RetAddr  Args to Child              
0860ee00 776febae 000015b0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15 (FPO: [3,0,0])
0860ee64 776fea92 00000000 00000000 008a27d8 ntdll!RtlpWaitOnCriticalSection+0x13e (FPO: [2,17,4])
0860ee8c 001c5279 008a2bd4 7a46e768 008be4b8 ntdll!RtlEnterCriticalSection+0x150 (FPO: [1,3,4])
0860eebc 00196118 00004338 0860eee0 7a46fa1c gssssvr!CBaseServer::GetRoomPtr+0x39
0860f3c8 00188131 00004338 0000004f 00000001 gssssvr!CMainServer::YQW_CheckOnline+0x58
0860f8dc 0018e168 14839778 141e5b70 00893fa0 gssssvr!CMainServer::OnGamePulse+0x141
0860f8f8 001f46ae 14839778 141e5b70 008a27d8 gssssvr!CMainServer::OnRequest+0x108
0860f910 00138320 14839778 141e5b70 7a46f094 gssssvr!CSkServer::OnRequest+0xae
0860f93c 002055c4 14839778 141e5b70 008a27e0 gssssvr!CGameServer::OnRequest+0x120 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 197]
0860f964 001f92bb 00000000 047601f0 047605b8 gssssvr!CIocpWorker::DoWorkLoop+0xa4
0860f97c 001f928b 0860f9bc 7194c01d 008a27d8 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
0860f984 7194c01d 008a27d8 d5a7fcff 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
0860f9bc 7194c001 00000000 0860f9d4 7623336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
0860f9c8 7623336a 047605b8 0860fa14 776e9902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
0860f9d4 776e9902 047605b8 d0a480b8 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
0860fa14 776e98d5 7194bfb4 047605b8 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
0860fa2c 00000000 7194bfb4 047605b8 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])

```
可以看出这八个线程都在等待一个临界区008a2bd4

### 3. 查看临界区008a2bd4被谁占有
输入命令：

```
!cs 008a2bd4
```
输出：

```
Critical section   = 0x008a2bd4 (+0x8A2BD4)
DebugInfo          = 0x008b4700
LOCKED
LockCount          = 0x11
WaiterWoken        = No
OwningThread       = 0x00000e24
RecursionCount     = 0x1
LockSemaphore      = 0x15B0
SpinCount          = 0x00000000
```
从上面输出可以看出0x008a2bd4被0x00000e24线程持有着，那么我们应该看看0x00000e24它到底发生了什么。

### 4. 查看0x00000e24是几号线程
输入命令：

```
~~[0x00000e24]
```
输出：

```
.  2  Id: 146c.e24 Suspend: 1 Teb: 7ef9c000 Unfrozen
      Start: msvcr120!_threadstartex (7194bfb4) 
      Priority: 0  Priority class: 32  Affinity: f
```

### 5. 查看2号线程的栈情况

输入命令：

```
~2s;kb
```
输出：

```
eax=00000000 ebx=00000000 ecx=00000000 edx=00000000 esi=13cf1138 edi=00000000
eip=776cf8e1 esp=02effa9c ebp=02effb00 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
ntdll!ZwWaitForSingleObject+0x15:
776cf8e1 83c404          add     esp,4
ChildEBP RetAddr  Args to Child              
02effa9c 776febae 000013a0 00000000 00000000 ntdll!ZwWaitForSingleObject+0x15
02effb00 776fea92 00000000 00000000 008a27d8 ntdll!RtlpWaitOnCriticalSection+0x13e
02effb28 001a5147 13cf1138 70c9f4e4 008a27d8 ntdll!RtlEnterCriticalSection+0x150
02effd30 001e3541 00000000 008bbce8 008b66c0 gssssvr!CMainServer::YQW_RoomPlayingTableTimeout+0x217
02effd54 001e347e 02effd94 7194c01d 008a27d8 gssssvr!CBaseServer::TimerThreadProc+0xb1
02effd5c 7194c01d 008a27d8 df28f8d7 00000000 gssssvr!CBaseServer::TimerThreadFunc+0xe
02effd94 7194c001 00000000 02effdac 7623336a msvcr120!_callthreadstartex+0x1b [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
02effda0 7623336a 008b66c0 02effdec 776e9902 msvcr120!_threadstartex+0x7c [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
02effdac 776e9902 008b66c0 da2b8740 00000000 kernel32!BaseThreadInitThunk+0xe
02effdec 776e98d5 7194bfb4 008b66c0 ffffffff ntdll!__RtlUserThreadStart+0x70
02effe04 00000000 7194bfb4 008b66c0 00000000 ntdll!_RtlUserThreadStart+0x1b
```

可以看出2号线程也在等待一个临界区，这个临界区是13cf1138

### 6. 查看临界区13cf1138被谁持有
输入命令：

```
!cs 13cf1138
```
输出：

```
Critical section   = 0x13cf1138 (+0x13CF1138)
DebugInfo          = 0x13c2f4c8
LOCKED
LockCount          = 0x1
WaiterWoken        = No
OwningThread       = 0x00000c68
RecursionCount     = 0x1
LockSemaphore      = 0x13A0
SpinCount          = 0x00000000
```

输入命令：

```
~~[0x00000c68]
```
输出：

```
53  Id: 146c.c68 Suspend: 1 Teb: 7ef03000 Unfrozen
      Start: msvcr120!_threadstartex (7194bfb4) 
      Priority: 0  Priority class: 32  Affinity: f
```

# 结案
53号线程持有临界区0x13c2f4c8，在等待临界区0x008a2bd4，
2号线程持有临界区0x008a2bd4，在等待临界区0x13c2f4c8，经典的互相等待死锁。。。接下来只要review代码即可。