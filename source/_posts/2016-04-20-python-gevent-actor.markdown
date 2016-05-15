---
layout: post
title: 扩展Python Gevent的Actor模型
comments: true
categories: [python, gevent]
---

## 什么是Actor模型？

[Actor模型](https://en.wikipedia.org/wiki/Actor_model)（[中文版](https://zh.wikipedia.org/wiki/%E5%8F%83%E8%88%87%E8%80%85%E6%A8%A1%E5%BC%8F)）是一种基于消息传递（message-passing）的并发（concurrent）计算模型。

它与OOP异同：

* 它推崇“一切皆为Actor”，而OOP推崇“一切皆为Object”
* 表面上，Actor通过发送消息与其他Actor通信，OOP的Object通过发送消息与其他Object通信。实际上，前者为发送结构化的数据，而后者为调用对方的方法。
* 它的发送者与已经发送的消息解耦，它允许进行异步通信，从而实现发送者与接收者并发执行。而OOP的方法调用者（发送者）与方法被调用者（接收者）通常顺序执行，而且调用者与被调用者通常具有较强的耦合。
* 它的消息接收者是通过地址区分的，有时也被称作“邮件地址”。而OOP的Object通过引用（地址）来区分。
* 它着重消息传递，而OOP着重于类与对象。

## Gevent的Actor实现

[gevent](http://sdiehl.github.io/gevent-tutorial/)（[中文版](http://xlambda.com/gevent-tutorial/)）是一个基于libev的并发库，它为各种并发和网络相关的任务提供了整洁的API。

其[Actors](http://sdiehl.github.io/gevent-tutorial/#actors)（[中文版](http://xlambda.com/gevent-tutorial/#actors)）章节已介绍了如何基于Greenlet和 Queue实现

该实现存在的问题：发送者与接收者紧耦合，发送者持有接收者的对象引用。

## 解决办法

在此基础上，我利用[message](http://blog.csdn.net/gzlaiyonghao/article/details/7215315)库将其扩展为发布-订阅者模式。

```python
from gevent.queue import Queue
from gevent import Greenlet
from message import observable


class Actor(Greenlet):
    def __init__(self):
        self.inbox = Queue()
        Greenlet.__init__(self)

    def send(self, message):
        self.inbox.put(message)

    def receive(self, message):
        raise NotImplemented()

    def _run(self):
        self.running = True

        while self.running:
            message = self.inbox.get()
            self.receive(message)


@observable
class Publisher(Actor):
    def __init__(self, subject):
        self.subject(subject)
        Actor.__init__(self)

    def subcribe(self, observer):
        self.sub(self.subject, observer.send)

    def publish(self, message):
        self.pub(self.subject, message)
```

如此，不仅将发送者与接收者解耦，而且支持发送者发送1条消息时，多个接收者接收同1条消息。
