---
layout: post
title: "线程异常退出导致死锁问题的发现与解决方法"
date: 2017-11-16
excerpt: "各种Memory"
tags: [线程, 死锁, C++]
comments: true
---

## 线程异常退出导致死锁
### 分析结果
指针偏移读到脏数据
-->json解析抛出异常
-->异常没有被捕获
-->线程异常退出
-->自动锁没有被正常释放
-->线程死锁

### 现象
服务器进程存在，但无响应

### 准备分析条件
1. 记录当时的服务的cpu占有率和内存占有率（任务管理器）
2. 在任务管理器中选中服务器进程右键创建转储文件（dmp文件）
3. 拿到服务器进程在构建过程中产生的pdb文件（一定要和发生问题的进程相对应，这样我们才能发现正确的代码行和堆栈）

### 开始分析
#### 先看进程cpu和内存状态

一切正常，可以排除内存泄漏还有计算型死循环
> io型死循环的cpu占用率其实不高

#### 什么都别说，直接看dmp

1. 打开windbg，将dmp文件拖入
2. 加载符号文件（进程对应的符号文件和微软操作系统的符号文件）

```
e:\yourpbdpath\;
e:\mylocalsymbols;
SRV*e:\mylocalsymbols*http://msdl.microsoft.com/download/symbols
```
3. 在命令框里面输入.reload命令重新加载pdb文件
> 这时候可能会提示你一些符号文件不存在，没关系，你可能只是少了微软的pdb，记得加上e:\mylocalsymbols;
SRV\*e:\mylocalsymbols\*http://msdl.microsoft.com/download/symbols然后重新reload就好。
4. 在命令框里输入!analyze -v命令对程序当时的状态进行分析
> 提示：输入命令后你可能等了好久没有响应，其实是正常的，不必当心输错了命令，关注一下命令框左边的状态\*busy\*说明命令正在执行中


5. 我们来看一下当时所有线程的情况，命令框中输入~*kv查看所有的线程，一下子输出了170条线程堆栈信息，一脸懵逼？没关系，我们只关心在等待临界区的线程和一些看起来像是自己（非系统）发起的线程。把这些可疑的线程堆栈摘出来。


```

  59  Id: ffc.c30 Suspend: 1 Teb: 7eeeb000 Unfrozen
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
0921f160 77d3ea92 00000000 00000000 00792f88 ntdll!NtWaitForSingleObject+0x15
0921f188 00e72663 045ab944 4072e9f3 045c4ba0 ntdll!RtlRandomEx+0x1ca
0921f690 00e650f1 00004005 00000001 00000000 gssssvr!CMainServer::YQW_CheckOnline+0xa3
0921fba4 00e6aa7e 09fd1080 09f043d8 0084eb20 gssssvr!CMainServer::OnGamePulse+0x141
0921fbc0 00ec82de 09fd1080 09f043d8 00792f88 gssssvr!CMainServer::OnRequest+0x10e
0921fbd8 00e1bcaa 09fd1080 09f043d8 4072e36b gssssvr!CSkServer::OnRequest+0xae
0921fc04 00ed9124 09fd1080 09f043d8 00792f90 gssssvr!CGameServer::OnRequest+0xfa [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 191]
0921fc2c 00eccf1b 00000000 045f8a88 045f6c48 gssssvr!CIocpWorker::DoWorkLoop+0xa4
0921fc44 00ecceeb 0921fc84 6f7ac01d 00792f88 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
0921fc4c 6f7ac01d 00792f88 97fb22b0 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
0921fc84 6f7ac001 00000000 0921fc9c 7696336a msvcr120!_get_flsindex+0x6f
0921fc90 7696336a 045f6c48 0921fcdc 77d29902 msvcr120!_get_flsindex+0x53
0921fc9c 77d29902 045f6c48 aab5016d 00000000 kernel32!BaseThreadInitThunk+0x12
0921fcdc 77d298d5 6f7abfb4 045f6c48 ffffffff ntdll!RtlInitializeExceptionChain+0x63
0921fcf4 00000000 6f7abfb4 045f6c48 00000000 ntdll!RtlInitializeExceptionChain+0x36

  58  Id: ffc.13c8 Suspend: 1 Teb: 7eeee000 Unfrozen
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
0906ed00 77d3ea92 00000000 00000000 00792f88 ntdll!NtWaitForSingleObject+0x15
0906ed28 00e72663 045ab944 4055ed53 09f7a058 ntdll!RtlRandomEx+0x1ca
0906f230 00e650f1 00004005 00000001 00000001 gssssvr!CMainServer::YQW_CheckOnline+0xa3
0906f744 00e6aa7e 09fd10e0 09fe0ea0 0084eb40 gssssvr!CMainServer::OnGamePulse+0x141
0906f760 00ec82de 09fd10e0 09fe0ea0 00792f88 gssssvr!CMainServer::OnRequest+0x10e
0906f778 00e1bcaa 09fd10e0 09fe0ea0 4055e8cb gssssvr!CSkServer::OnRequest+0xae
0906f7a4 00ed9124 09fd10e0 09fe0ea0 00792f90 gssssvr!CGameServer::OnRequest+0xfa [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 191]
0906f7cc 00eccf1b 00000000 045f86c0 008295e8 gssssvr!CIocpWorker::DoWorkLoop+0xa4
0906f7e4 00ecceeb 0906f824 6f7ac01d 00792f88 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
0906f7ec 6f7ac01d 00792f88 97dc2610 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
0906f824 6f7ac001 00000000 0906f83c 7696336a msvcr120!_get_flsindex+0x6f
0906f830 7696336a 008295e8 0906f87c 77d29902 msvcr120!_get_flsindex+0x53
0906f83c 77d29902 008295e8 aa9205cd 00000000 kernel32!BaseThreadInitThunk+0x12
0906f87c 77d298d5 6f7abfb4 008295e8 ffffffff ntdll!RtlInitializeExceptionChain+0x63
0906f894 00000000 6f7abfb4 008295e8 00000000 ntdll!RtlInitializeExceptionChain+0x36

  57  Id: ffc.438 Suspend: 1 Teb: 7eef1000 Unfrozen
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
08ebe2ac 77d3ea92 00000000 00000000 00792f90 ntdll!NtWaitForSingleObject+0x15
08ebe2d4 00dd2031 045ab944 045ab944 08ebe2f0 ntdll!RtlRandomEx+0x1ca
08ebe2e4 00dd1789 08ebe4bc 08ebf964 00e1cfa5 gssssvr!CCritSec::Lock+0x11 [d:\svn143\library\uwl9.1\uwlutil.h @ 137]
08ebe2f0 00e1cfa5 045ab944 41b8e607 09f0e450 gssssvr!CAutoLock::CAutoLock+0x19 [d:\svn143\library\uwl9.1\uwlutil.h @ 160]
08ebf964 00e1bc40 007c5d30 09f04298 0084eb50 gssssvr!CGameServer::OnSendSysMsgToServer+0x385 [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 571]
08ebf994 00ed9124 007c5d30 09f04298 00792f90 gssssvr!CGameServer::OnRequest+0x90 [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 162]
08ebf9bc 00eccf1b 00000000 045f82f8 045f6c48 gssssvr!CIocpWorker::DoWorkLoop+0xa4
08ebf9d4 00ecceeb 08ebfa14 6f7ac01d 00792f88 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
08ebf9dc 6f7ac01d 00792f88 96312420 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
08ebfa14 6f7ac001 00000000 08ebfa2c 7696336a msvcr120!_get_flsindex+0x6f
08ebfa20 7696336a 045f6c48 08ebfa6c 77d29902 msvcr120!_get_flsindex+0x53
08ebfa2c 77d29902 045f6c48 ab7f07dd 00000000 kernel32!BaseThreadInitThunk+0x12
08ebfa6c 77d298d5 6f7abfb4 045f6c48 ffffffff ntdll!RtlInitializeExceptionChain+0x63
08ebfa84 00000000 6f7abfb4 045f6c48 00000000 ntdll!RtlInitializeExceptionChain+0x36

  55  Id: ffc.14b8 Suspend: 1 Teb: 7eef7000 Unfrozen
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
08ade6ac 77d3ea92 00000000 00000000 00792f90 ntdll!NtWaitForSingleObject+0x15
08ade6d4 00dd2031 045ab944 045ab944 08ade6f0 ntdll!RtlRandomEx+0x1ca
08ade6e4 00dd1789 08ade8bc 08adfd64 00e1cfa5 gssssvr!CCritSec::Lock+0x11 [d:\svn143\library\uwl9.1\uwlutil.h @ 137]
08ade6f0 00e1cfa5 045ab944 41fee207 09fe0d88 gssssvr!CAutoLock::CAutoLock+0x19 [d:\svn143\library\uwl9.1\uwlutil.h @ 160]
08adfd64 00e1bc40 09fd10a0 09f04270 0084eb70 gssssvr!CGameServer::OnSendSysMsgToServer+0x385 [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 571]
08adfd94 00ed9124 09fd10a0 09f04270 00792f90 gssssvr!CGameServer::OnRequest+0x90 [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 162]
08adfdbc 00eccf1b 00000000 045f7b68 045f6c48 gssssvr!CIocpWorker::DoWorkLoop+0xa4
08adfdd4 00ecceeb 08adfe14 6f7ac01d 00792f88 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
08adfddc 6f7ac01d 00792f88 96772020 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
08adfe14 6f7ac001 00000000 08adfe2c 7696336a msvcr120!_get_flsindex+0x6f
08adfe20 7696336a 045f6c48 08adfe6c 77d29902 msvcr120!_get_flsindex+0x53
08adfe2c 77d29902 045f6c48 ab3903dd 00000000 kernel32!BaseThreadInitThunk+0x12
08adfe6c 77d298d5 6f7abfb4 045f6c48 ffffffff ntdll!RtlInitializeExceptionChain+0x63
08adfe84 00000000 6f7abfb4 045f6c48 00000000 ntdll!RtlInitializeExceptionChain+0x36

.......

```


6. 看看这些摘出来的线程堆栈还是有所收获的，52,53,54,55，（就是没有56）,57,58,59号线程都在等待同一个临界区（045ab944 ）

那么我是怎么找到这个临界区的呢？举个栗子，看下面这个55号线程堆栈
```
  55  Id: ffc.14b8 Suspend: 1 Teb: 7eef7000 Unfrozen
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
08ade6ac 77d3ea92 00000000 00000000 00792f90 ntdll!NtWaitForSingleObject+0x15
08ade6d4 00dd2031 045ab944 045ab944 08ade6f0 ntdll!RtlRandomEx+0x1ca
08ade6e4 00dd1789 08ade8bc 08adfd64 00e1cfa5 gssssvr!CCritSec::Lock+0x11 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\library\uwl9.1\uwlutil.h @ 137]
08ade6f0 00e1cfa5 045ab944 41fee207 09fe0d88 gssssvr!CAutoLock::CAutoLock+0x19 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\library\uwl9.1\uwlutil.h @ 160]
08adfd64 00e1bc40 09fd10a0 09f04270 0084eb70 gssssvr!CGameServer::OnSendSysMsgToServer+0x385 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 571]
08adfd94 00ed9124 09fd10a0 09f04270 00792f90 gssssvr!CGameServer::OnRequest+0x90 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 162]
08adfdbc 00eccf1b 00000000 045f7b68 045f6c48 gssssvr!CIocpWorker::DoWorkLoop+0xa4
08adfdd4 00ecceeb 08adfe14 6f7ac01d 00792f88 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
08adfddc 6f7ac01d 00792f88 96772020 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
08adfe14 6f7ac001 00000000 08adfe2c 7696336a msvcr120!_get_flsindex+0x6f
08adfe20 7696336a 045f6c48 08adfe6c 77d29902 msvcr120!_get_flsindex+0x53
08adfe2c 77d29902 045f6c48 ab3903dd 00000000 kernel32!BaseThreadInitThunk+0x12
08adfe6c 77d298d5 6f7abfb4 045f6c48 ffffffff ntdll!RtlInitializeExceptionChain+0x63
08adfe84 00000000 6f7abfb4 045f6c48 00000000 ntdll!RtlInitializeExceptionChain+0x36

```
第一行主要是这个线程的基本信息，线程号55，线程id：ffc.14b8。。。
第二行是三个表头，意思是函数入口地址，返回地址和函数参数，用于解释下面每行的五个十六进制数据（第一个是入口地址，第二个是返回地址，第三到第五个是函数参数）

再举个小栗子：
我们来看上面线程堆栈中的第四行
```
08ade6f0 00e1cfa5 045ab944 41fee207 09fe0d88 gssssvr!CAutoLock::CAutoLock+0x19 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\svn143\library\uwl9.1\uwlutil.h @ 160]

```
这是一个自动锁（CAutoLock），08ade6f0是它的入口地址，00e1cfa5是它的返回地址，045ab944是它的第一个参数。
> CAutoLock只有一个参数，临界区，所以045ab944就是这个线程要等待的临界区。

7. 思考——7个线程同时在等待一个临界区，十有八九是死锁了。第一个想法是会不会是线程之间互锁了，A线程持有了锁lockA在等待lockB，但是B线程持有了锁lockB在等待lockA，我看了下摘出的线程堆栈，找了半天，并没有啊。
8. 陷入焦虑——看源码如大海捞针，看堆栈又一筹莫展。
9. 新发现——56号线程抛出了一个异常，而通过源码发现服务器进程其实一次起了8个线程用于消息接收，但是我们只发现了7个线程在等待临界区。所以56号线程就是这个“消失”的消息接收线程。
> 小知识：一起创建的线程，他们的线程号是连续的，所以52,53,54,55,（56）,57,58,59基本可以确定，56号就是消息接收线程。
10. 确认——既然临界区（045ab944）被7个线程同时等待，那我们来查一查到底是哪个线程占有了这个线程，命令框里输入dt RTL_CRITICAL_SECTION 045ab944

输出如下信息：

```
gssssvr!RTL_CRITICAL_SECTION
   +0x000 DebugInfo        : 0x00842ce0 _RTL_CRITICAL_SECTION_DEBUG
   +0x004 LockCount        : -30
   +0x008 RecursionCount   : 1
   +0x00c OwningThread     : 0x000004d0 
   +0x010 LockSemaphore    : 0x00001238 
   +0x014 SpinCount        : 0

```
说明该临界区被0x000004d0线程占有（就是56号线程！！！！）

11. 56号到底发生了什么？先看堆栈输出56号线程的堆栈情况：

```
ChildEBP RetAddr  Args to Child              
08ccd8d4 76ddd846 00000000 00000000 00000000 user32!NtUserWaitMessage+0x15 (FPO: [0,0,0])
08ccd910 76ddda5c 06f400d2 00000000 00000000 user32!DialogBox2+0x222 (FPO: [4,10,4])
08ccd93c 76e0f7d0 76da0000 09f6fe90 00000000 user32!InternalDialogBox+0xe5 (FPO: [6,2,4])
08ccd9f0 76e0faac 00012010 00000000 ffffffff user32!SoftModalMessageBox+0x757 (FPO: [1,34,4])
08ccdb48 76e0fbaf 08ccdb54 00000028 00000000 user32!MessageBoxWorker+0x269 (FPO: [1,80,4])
08ccdbb4 76e0fc2e 00000000 0084acf8 09ee8d38 user32!MessageBoxTimeoutW+0x52 (FPO: [6,24,0])
08ccdbe8 76e0fd81 00000000 08ccdd84 0028f764 user32!MessageBoxTimeoutA+0x76 (FPO: [6,2,4])
08ccdc08 76e0fdc6 00000000 08ccdd84 0028f764 user32!MessageBoxExA+0x1b (FPO: [5,0,0])
*** WARNING: Unable to verify checksum for XYSoapClient.dll
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for XYSoapClient.dll - 
08ccdc24 0028cdd9 00000000 08ccdd84 0028f764 user32!MessageBoxA+0x18 (FPO: [4,0,0])
WARNING: Stack unwind information not available. Following frames may be wrong.
08ccde24 0028bf49 0000000a 00289e54 0028ba37 XYSoapClient!DllUnregisterServer+0x6bae
08ccde5c 769a03bb 08ccdf14 9616d04a 00000000 XYSoapClient!DllUnregisterServer+0x5d1e
08ccdee4 77d65be7 08ccdf14 77d65ac4 00000000 kernel32!UnhandledExceptionFilter+0x127 (FPO: [SEH])
08ccdeec 77d65ac4 00000000 08ccfe1c 77d1c620 ntdll!__RtlUserThreadStart+0x62 (FPO: [SEH])
08ccdf00 77d65951 00000000 00000000 00000000 ntdll!_EH4_CallFilterFunc+0x12 (FPO: [Uses EBP] [0,0,4])
08ccdf28 77d53529 fffffffe 08ccfe0c 08cce064 ntdll!_except_handler4+0x8e (FPO: [4,5,4])
08ccdf4c 77d534fb 08cce014 08ccfe0c 08cce064 ntdll!ExecuteHandler2+0x26 (FPO: [Uses EBP] [5,3,1])
08ccdf70 77d5349c 08cce014 08ccfe0c 08cce064 ntdll!ExecuteHandler+0x24 (FPO: [5,0,3])
08ccdffc 77d00143 01cce014 08cce064 08cce014 ntdll!RtlDispatchException+0x127 (FPO: [2,25,4])
08ccdffc 00000000 01cce014 08cce064 08cce014 ntdll!KiUserExceptionDispatcher+0xf (FPO: [2,0,0]) (CONTEXT @ 00000008)

```
有异常抛出，并尝试弹出一个对话框，一个服务器进程尝试显示对话框（GUI）显然是不合理的。KiUserExceptionDispatcher是异常分发函数，KiUserExceptionDispatcher 函数第一个参数为异常类型，第二个参数为产生异常时的上下文记录。


```
08ccdffc 00000000 01cce014 08cce064 08cce014 ntdll!KiUserExceptionDispatcher+0xf (FPO: [2,0,0]) (CONTEXT @ 00000008)
```

第二个参数08cce064就是当时的上下文。

12. 查看异常01cce014，命令框里输入.exr 01cce014

```
0:056> .exr 01cce014
Cannot read Exception record @ 01cce014
```
发现并不能看到什么

13. 恢复当时的上下文08cce064到寄存器，在命令框里面输入.cxr 08cce064

```
0:056> .cxr 08cce064
eax=08cce4c8 ebx=00792f90 ecx=00000003 edx=00000000 esi=00f3de44 edi=08cce570
eip=7767c54f esp=08cce4c8 ebp=08cce518 iopl=0         nv up ei pl nz ac po nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000212
*** ERROR: Symbol file could not be found.  Defaulted to export symbols for KERNELBASE.dll - 
KERNELBASE!RaiseException+0x58:
7767c54f c9              leave
```
然后使用kv命令恢复当时正确的堆栈如下：

```
0:056> kv
  *** Stack trace for last set context - .thread/.cxr resets it
ChildEBP RetAddr  Args to Child              
WARNING: Stack unwind information not available. Following frames may be wrong.
08cce518 6f799339 e06d7363 00000001 00000003 KERNELBASE!RaiseException+0x58
*** WARNING: Unable to verify checksum for gssssvr.exe
08cce558 00ee5f9b 08cce570 00f3de44 419ffac3 msvcr120!_CxxThrowException+0x5b (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\eh\throw.cpp @ 152]
08cce5a0 00ee3852 08cce660 419ff9eb 00f1be7d gssssvr!Json::throwLogicError+0x7b
08cce688 00ee42c4 00f1be7c 00f1be83 00792f88 gssssvr!Json::Value::find+0x82
08cce6a0 00e1d5a5 00f1be7c 419fe277 09f03cd0 gssssvr!Json::Value::isMember+0x24
08ccfd14 00e1bc40 007c5b50 09f04018 0084eb60 gssssvr!CGameServer::OnSendSysMsgToServer+0x985 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 675]
08ccfd44 00ed9124 007c5b50 09f04018 00792f90 gssssvr!CGameServer::OnRequest+0x90 (FPO: [Non-Fpo]) (CONV: thiscall) [d:\program files (x86)\jenkins\workspace\publish_gssssvr\gssssvr\server.cpp @ 162]
08ccfd6c 00eccf1b 00000000 045f7f30 008295e8 gssssvr!CIocpWorker::DoWorkLoop+0xa4
08ccfd84 00ecceeb 08ccfdc4 6f7ac01d 00792f88 gssssvr!CBaseWorker::WorkerThreadProc+0x2b
08ccfd8c 6f7ac01d 00792f88 961623f0 00000000 gssssvr!CBaseWorker::WorkerThreadFunc+0xb
08ccfdc4 6f7ac001 00000000 08ccfddc 7696336a msvcr120!_callthreadstartex+0x1b (FPO: [Non-Fpo]) (CONV: cdecl) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 376]
08ccfdd0 7696336a 008295e8 08ccfe1c 77d29902 msvcr120!_threadstartex+0x7c (FPO: [Non-Fpo]) (CONV: stdcall) [f:\dd\vctools\crt\crtw32\startup\threadex.c @ 354]
08ccfddc 77d29902 008295e8 ab5803ad 00000000 kernel32!BaseThreadInitThunk+0xe (FPO: [1,0,0])
08ccfe1c 77d298d5 6f7abfb4 008295e8 ffffffff ntdll!__RtlUserThreadStart+0x70 (FPO: [SEH])
08ccfe34 00000000 6f7abfb4 008295e8 00000000 ntdll!_RtlUserThreadStart+0x1b (FPO: [2,2,0])
```
直接定位到了抛出异常的代码，666，一个json解析抛出了异常，那么接下来只要看看代码为什么会抛出异常？

14. review代码，发现代码有一个非常危险的操作，在收到客户端的数据的时候，json数据是拼接在发送过来的结构体后面的，所以用了指针偏移操作，一旦客户端没有拼接json数据指针就会读到脏数据。json解析就会抛出异常，然后代码并没有捕获这个异常，这个线程就异常退出，持有的临界区就没有被正常释放，GG思密达。




