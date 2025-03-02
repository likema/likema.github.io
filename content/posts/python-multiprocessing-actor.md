---
title: 基于Python multiprocessing的Actor模型
date: 2016-07-11
categories: [Python, Actor]
---

虽然[基于Gevent的Actor](/blog/2016/04/20/python-gevent-actor/)和[基于Python 3.5异步的Actor](/blog/2016/07/03/python-3-dot-5-async-actor/)都支持并发（concurrent）计算（仅运行于单进程中），但是不支持并行（parallel）计算，即无法利用多核。

Python内置的multiprocessing模块不仅支持并行计算，而且与Gevent接口相似。所以，模仿Gevent的Actor实现multiprocessing的Actor并不困难。

## multiprocessing的Actor实现

```python
from multiprocessing import Process, Queue

try:
    from Queue import Empty
except ImportError:
    from queue import Empty


class Actor(Process):
    def __init__(self, receive_timeout=None):
        Process.__init__(self)
        self.inbox = Queue()
        self.receive_timeout = receive_timeout

    def send(self, message):
        self.inbox.put_nowait(message)

    def receive(self, message):
        raise NotImplemented()

    def handle_timeout(self):
        pass

    def run(self):
        self.running = True
        while self.running:
            try:
                message = self.inbox.get(True, self.receive_timeout)
            except Empty:
                self.handle_timeout()
            else:
                self.receive(message)
```

## 基于message的扩展

将并行Actor扩展为发布-订阅者模式，基本与Gevent的实现一样。

```python
from message import observable

from actor import Actor


@observable
class Publisher(Actor):
    def __init__(self, subject, receive_timeout=None):
        self.subject = subject
        Actor.__init__(self, receive_timeout)

    def subcribe(self, observer):
        self.sub(self.subject, observer.send)

    def publish(self, message):
        self.pub(self.subject, message)
```

基于Publisher实现Ping-Pong，与Gevent的实现差异也不大。

不同的是它实际启动3个进程。除主进程外，每个actor分别运行于独立进程，从而实现多核计算。主进程监督2个actor进程运行，如启动、停止以及异常处理等。

```python
import time

from publisher import Publisher


class Pinger(Publisher):
    def receive(self, message):
        print(message)
        time.sleep(2)
        self.publish('ping')

    def handle_timeout(self):
        print('pinger timeout')


class Ponger(Publisher):
    def receive(self, message):
        print(message)
        time.sleep(2)
        self.publish('ping')

    def handle_timeout(self):
        print('ponger timeout')


ping = Pinger('evt.ping', 1)
pong = Ponger('evt.pong', 1)

ping.subcribe(pong)
pong.subcribe(ping)
ping.start()
pong.start()

ping.publish('start')

pong.join()
ping.join()
```

