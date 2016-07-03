---
layout: post
title: 基于Python 3.5异步的Actor模型
comments: true
categories: [Python, Actor]
---

## Python 3.5异步模型

Python 3.5推出了async/await语法，在语法层面简化了异步编程。官方库asyncio是应用async/await的途径。

Ubuntu 16.04默认安装Python 3.5，或者通过[pyenv](/blog/2016/05/21/pyenv-tutorial/)安装它。

## 异步Actor的实现

基于asyncio，可以实现async actor.

```python
import asyncio


class Actor(object):
    def __init__(self):
        self.inbox = asyncio.Queue()
        
    def send(self, message):
        self.inbox.put_nowait(message)

    async def receive(self, message):
        raise NotImplemented()

    async def run(self):
        self.running = True

        while self.running:
            message = await self.inbox.get()
            await self.receive(message)
```

上述代码的关键是通过asyncio.Queue异步接收消息，并异步处理接收到的消息。

通过这个类，实现Ping-Pong示例：

```python
import asyncio

from actor import Actor


class Pinger(Actor):
    async def receive(self, message):
        print(message)
        pong.send('ping')
        await asyncio.sleep(3)


class Ponger(Actor):
    async def receive(self, message):
        print(message)
        ping.send('pong')
        await asyncio.sleep(3)


ping = Pinger()
pong = Ponger()

ping.send('start')

loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait((ping.run(), pong.run())))
loop.close()
```

该示例代码中，actor之间同步发送消息（asyncio.Queue.put\_nowait），由于运行在单线程上，并不存在竞争。

## 接收消息超时(timeout)

某些应用场景需要周期性激活Actor，当Actor没有收到任何消息时。

基于上述代码，利用asyncio.wait\_for的超时功能来实现接收消息超时。如此，进一步加强Actor的并发能力。

```python
import asyncio


class Actor(object):
    def __init__(self, receive_timeout=None):
        self.inbox = asyncio.Queue()
        self.receive_timeout = receive_timeout
        
    def send(self, message):
        self.inbox.put_nowait(message)

    async def receive(self, message):
        raise NotImplemented()

    async def handle_timeout(self):
        pass

    async def run(self):
        self.running = True

        while self.running:
            try:
                message = await asyncio.wait_for(self.inbox.get(),
                                                 self.receive_timeout)
            except asyncio.TimeoutError:
                await self.handle_timeout()
            else:
                await self.receive(message)
```

## 基于message的扩展

由于[message](http://blog.csdn.net/gzlaiyonghao/article/details/7215315)仅支持Python 2，而且Google Code已经停止服务。

基于原代码基础上，我在GitHub创建[python-message](https://github.com/likema/python-message)，并扩展支持Python 3.

新版本message，也可以通过pip安装：

```bash
pip install https://github.com/likema/python-message/archive/master.zip
```

在此基础上，将异步Actor扩展为发布-订阅者模式。

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

基于Publisher实现Ping-Pong，从而解耦发送者与接收者，且支持发送者发送1条消息时，多个接收者接收同1条消息。

```python
import asyncio

from publisher import Publisher


class Pinger(Publisher):
    async def receive(self, message):
        print(message)
        await asyncio.sleep(3)
        self.publish('ping')

    async def handle_timeout(self):
        print('Pinger timeout')


class Ponger(Publisher):
    async def receive(self, message):
        print(message)
        await asyncio.sleep(3)
        self.publish('pong')

    async def handle_timeout(self):
        print('Ponger timeout')


ping = Pinger('evt.ping', 1)
pong = Ponger('evt.pong', 1)

ping.subcribe(pong)
pong.subcribe(ping)
ping.publish('start')

loop = asyncio.get_event_loop()
loop.run_until_complete(asyncio.wait((ping.run(), pong.run())))
loop.close()
```

## 存在的问题

相比[Gevent实现的Actor](/blog/2016/04/20/python-gevent-actor/)，异步Actor并不透明支持所有的I/O函数，它仅支持基于asyncio实现的库，如[aiohttp](http://aiohttp.readthedocs.io/en/stable/)。

## 参考

* [Python async/await Tutorial](http://stackabuse.com/python-async-await-tutorial/)
* [python3.5-async-crawler](https://github.com/mehmetkose/python3.5-async-crawler)



