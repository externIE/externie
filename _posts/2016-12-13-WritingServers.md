---
layout: post
title: "写一个twisted服务器"
date: 2016-12-13
excerpt: "关于Twisted server看这篇就够了"
tags: [python, twisted, twistd]
comments: true
---

## 概要

这篇文章将向你展示怎么用twisted框架去实现网络协议的解析和操作TCP Server（代码可被SSL和Unix socket servers重用）。
你关于协议操作的class必须继承于twisted.internet.protocol.Protocol。大部分的protocol handler继承于这个类或者它的子类。
一个protocol类的实例可以按需实例化每个connection，并且当connection结束的时候会自动销毁。这意味着持续性的配置不会在protocol里面被保存更新。
持续性的配置应该在Factory类里面进行维护，Factory类一般都继承至twisted.internet.protocol.Factory。Factory的buildProtocol方法被用来创建
每个connection的protocol。
通常为一个server提供不同的ip地址和端口是非常有用的。这就是为什么Factory不会去监听connection，事实上factory一点都不懂网络。

## Protocols

twisted protocol是异步处理数据的。protocal响应从网络上到达的事件和内部函数调用的事件。
下面是一个简单的例子🌰：
{% highlight python %}
from twisted.internet.protocol import Protocol 
class Echo(Protocol):
    def dataReceived(self, data):
        self.transport.write(data)
{% endhighlight %}
这是最简单的一个protocol。它只是简单的将所有收到的数据写回去，并不响应任何事件。
下面是相应另外一个事件的例子🌰：
{% highlight python %}
from twisted.internet.protocol import Protocol
class QOTD(Protocol):
    def connectionMade(self):
        self.transport.write("this is a test")
        self.transport.loseConnection()
{% endhighlight %}
上面这个protocol在一个初始化的connection中返回了“this is a test”的文字，并且挂起了connection。
connectionMade事件通常是在设置connection对象的时候发生，跟其他的初始化方法一样（就像QOTD protocol基于RFC865）。
connectionLost事件通常在任何特殊的connection对象结束的被调用。
下面是再举个例子🌰：
{% highlight python %}
from twisted.internet.protocol import Protocol
class Echo(Protocol):
    def __init__(self, factory):
        self.factory = factory
    def connectionMade(self):
        self.factory.numProtocols = self.factory.numProtocols + 1
        self.transport.write(
            "welcome! there are currently %d open connections.\n"%
            (self.factory.numProtocols)
        )
    def connectionLost(self, reason):
        self.factory.numProtocols = self.factory.numProtocols - 1
    def dataReceived(self, data):
        self.transport.write(data)
{% endhighlight %}
上面的connectionMade和connectionLost在一个共享变量factory中协同维护了一个关于活跃protocol的计数。
factory对象必须在初始化的时候传给echo。factory被用来共享状态并不在任何connection的周期中。你将会在下个单元看到为什么这个对象要叫“factory”。

## loseConnection()和abortConnection()

在上面的代码中，loseConnection立马就被调用了在transports中写入信息之后。loseConnection只会在twisted将所有的数据都写出到操作系统才会关闭，
所以在这里调用这个函数是安全的，你不用当心transports写入的数据会丢失。如果producer正在使用transport的话loseConnection只会关闭connection当producer没有注册的时候。
某些情况下，等所有数据都写出并不是我们想要的。比如由于connection的一端的网络问题或者bug问题导致，transport不能正常的实现数据的传递。这时候即便loseConnection被调用
也不能有效果，因为还有部分数据没有写出，loseConnection就没法关闭。这种情况，我们就可以用到abortConnection来立即强制关闭所有的connection，不考虑缓冲中的数据还没有写出，
或者producer还是处于注册状态。

**注意!** abortConnection只能用于twisted 11.1及以上的版本
{: .notice}

## 使用Protocol

在这个单元，你将学会在你的server中用你写的Protocol。
下面这段代码将会用到之前讨论过的QOTD
{% highlight python %}
from twisted.internet.protocol import Factory 
from twisted.internet.endpoints import TCP4ServerEndpoint
from twisted.internet import reactor

class QOTDFactory(Factory):
    def buildProtocol(self, addr):
        return QOTD()
# 8007 是你的端口(大于1024)
endpoint = TCP4ServerEndpoint(reactor, 8007)
endpoint.listen(QOTDFactory())
reactor.run()
{% endhighlight %}
在这个例子中，我们创建了protocol Factory。Factory的作用就是创建QOTD protocol的实例，所以我们在buildProtocol中返回QOTD类的实例。
然后我们希望能够监听TCP端口，所以我用TCP4ServerEndpoint指明了我们要绑定的端口，然后把QOTDFactory传进去。
endpoint.listen()告诉reactor用指定的端口连接endpoint的地址。reactor.run()开始了事件循环并一直等待端口的connection的到来。
你可以停止通过快捷键ctr+c或reactor.stop()来停止reactor。
不同的监听方式你可以看 [关于endpoints的API](https://twistedmatrix.com/documents/current/core/howto/endpoints.html)。
更多关于reactor的资料你可以看 [关于Reactor](https://twistedmatrix.com/documents/current/core/howto/reactor-basics.html)。


## Helper Protocol

许多的protocol是建立在低级抽象之上。
例如，许多流行的internet protocol（英特网协议）是以行为单位的数据，通过换行符（一般来说是CR-LF）来实现行的截断，而不是直接提供raw data（原始数据）。
然而，还是有不少protocol是混合型的，它们既有以行为单位，也有以原始数据为单位。比如HTTP/1.1和Freenet协议。
混合型的protocol有[LineReceiver](https://twistedmatrix.com/documents/16.5.0/api/twisted.protocols.basic.LineReceiver.html)。
这个协议有两个不同的event handler--lineReceived和rawDataReceived。默认情况下，只有lineReceived才会被调用，一次一行。但是如果setRawMode被调用，
protocol就会调用rawDataReceived除非setLineMode被调用才会重新启用lineReceived。同时它也提供一个方法，sendLine，用分隔符（默认为\r\n）把数据分割写入transport。
下面是一个line receiver的简单应用例子🌰：
{% highlight python %}
from twisted.protocols.basic import LineReceiver
class Answer(LineReceiver):
    answers = {'How are you?':'Fine', None:'I don't know what you mean'}
    def LineReceived(self, line):
        if line in self.answers:
            self.sendLine(self.answers[line])
        else:
            self.sendLine(self.answers[None])
{% endhighlight %}

***注意！***分隔符并不属于内容
{: .notice}
还有许多的helper存在，比如说[netstring based protocol](https://twistedmatrix.com/documents/16.5.0/api/twisted.protocols.basic.NetstringReceiver.html)和
[prefixed-message-length protocols](https://twistedmatrix.com/documents/16.5.0/api/twisted.protocols.basic.IntNStringReceiver.html)

## 状态机

许多twisted protocol handler需要一个状态机来记录它们的状态。下面是一些帮助你写状态机的建议：
* 别写很庞大的状态机。最好的状态机是只有一层抽象。
* 别把应用相关的代码和protocol handler的代码混在一起。当protocol handler必须调用应用方代码的时候，尽量用方法形式调用。

## Factories

### 简单的protocol创建
factory其实就是用来实例化一些指定的protocol类，实现factory的方法其实很简单。

## 未完待续。。。