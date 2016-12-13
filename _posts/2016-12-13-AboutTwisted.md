---
layout: post
title: "关于Twisted的很多事"
date: 2016-12-13
excerpt: "关于Twisted看这篇就够了"
tags: [python, twisted, twistd]
comments: true
---

## Twisted 概述
最近博主经历了一些事，话说生活上的事真的是说不清道不明，许多事都超出了我的处理能力范围，我不知道该怎么去解决，但也不能放着，因为有那么一刻我觉得，
有些事即便做了也是没有价值的，但你还是得义无反顾，要不然可能后悔一辈子。生活太复杂，相比下代码要优雅的多，我们还是来义无反顾的学学twisted吧。
扯了那么多，那么什么是twited呢？
twisted其实就是一个事件驱动型的网络引擎，好像没什么亮点，可能是我还没告诉你它是用python写的。
那么有个问题，到底什么是事件驱动模型？

### 事件驱动模型
事件驱动模型其实本质上是属于单线程模型的，当你在执行一些代价比较大的I／O操作的时候，你可以注册一个回调到事件循环中，当事件结束时会调用你注册的回调。
事件循环会不断的轮询所有注册的事件，当事件到来时就会调用相应的事件回调。由于是单线程的，所以你不用考虑共享数据和线程安全的问题。

### 什么时候适合事件驱动模型呢
* 当很多任务之间彼此高度独立的时候
* 某些任务会阻塞等待事件的到来
* 任务之间要共享一些动态数据的时候（单线程嘛）

### 举个栗子🌰
早上起来，先开电饭煲煮饭（任务A），再去刷牙洗脸（任务B），洗漱完后炒菜（任务C，任务B的回调），电饭煲提示你饭好了（事件到达，回调吃饭），吃饭，然后下一次循环。。。

## 代码实例

### Echo Server
twisted可以很容易的实现一个网络应用。下面是一个TCP Server的示例代码：
<% highlight python %>
from twisted.internet import protocol, reactor, endpoints
class Echo(protocol.Protocol):
    def dataReceived(self, data):
        self.transport.write(data)

class EchoFactory(protocol.Factory):
    def buildProtocol(self, addr):
        return Echo()

endpoints.serverFromString(reactor, "tcp:1234").listen(EchoFactory())
reactor.run()
<% endhighlingt%>

### Web Server
twisted包括一个事件驱动的web server，下面是一个简单的网络应用
**注意!** resource对象一直存在于内存中，不需要每个request都去重复创建
{: .notice}
<% highlight python %>
from twisted.web import server, resource
from twisted.internet import reactor, endpoints

class Counter(resource.Resource):
    isLeaf = True
    numberRequests = 0
    def render_GET(self, request):
        self.numberRequests += 1
        request.setHeader(b"content-type", b"text/plain")
        content = u"I am request #{}\n".format(self.numberRequests)
        return content.encode("ascii")

endpoints.serverFromString(reactor, "tcp:8080").listen(server.Site(Counter()))
<% endhighlingt %>

### 发布/订阅服务器
这是一个简单的发布/订阅服务器，客户端可以见到所有服务器发布的消息
<% highlight python %>
from twisted.internet import reactor, protocol, endpoints
from twisted.protocol import basic
class PubProtocol(basic.LineReceiver):
    def __init__(self, factory):
        self.factory = factory
    def connectionMade(self):
        self.factory.clients.add(self)
    def connectionLost(self, reason):
        self.factory.clients.remove(self)
    def lineReceiver(self, line):
        for c in self.factory.clients:
            source = u"<{}>".format(self.transport.getHost()).encode("ascii")
            c.sendLine(source + line)

class PubFactory(protocol.Factory):
    def __init__(self):
        self.clients = set()
    def buildProtocol(self, addr):
        return PubProtocol(self)

endpoints.serverFromString(reactor, "tcp:1025").listen(PubFactory())
reactor.run()
<% endhighlingt %>

### 邮件客户端
twisted包含了一个丰富的IMAP4客户端library
<% highlight python %>
from __future__ import print_function
import sys
from twisted.internet import protocol, defer, endpoints, task
from twisted.mail import imap4
from twisted.python import failure
@defer.inlineCallbacks
def main(reactor, username=="alice", password="secret", strport="ssl:host=example.com:port=993"):
    endpoints = endpoints.clientFromString(reactor, strport)
    factory = protocol.Factory.forProtocol(imap4.IMAP4Client)
    try:
        client = yield endpoints.connect(factory)
        yield client.login(username, password)
        yield client.select("INBOX")
        info = yield client.fetchEnvelope(imap4.MessageSet(1))
        print("First message subject:", info[1]['ENVELOPE'][1])
    except:
        print("IMAP4 client interaction failed")
        failure.Failure().printTraceback()
task.react(main, sys.argv[1:])
<% endhighlingt %>
