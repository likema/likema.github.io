---
title: 用ACE实现One Loop Per Thread
date: 2020-04-19 19:20:02
categories: [C++, ACE]
---

[one loop per thread](https://www.cnblogs.com/Solstice/archive/2010/02/12/multithreaded_server.html#_Toc7583) 即每个线程运行一个独立的事件循环 (event loop)，或称为 one event loop per thread 更准确。而且事件循环线程之间通常不存在同步或互斥的关系, 并发连接的处理能力随CPU核心数而扩展 (scalable)。

[ACE](http://www.dre.vanderbilt.edu/~schmidt/ACE.html) 拥有多种实现的Reactor:

* `ACE_Select_Reactor` : 基于 `select` 的实现。
* `ACE_Dev_Poll_Reactor` : 基于 Linux `epoll` 或 BSD `/dev/poll` 的实现。
* `ACE_WFMO_Reactor` : 基于 Windows `WaitForMultipleObjects` 的实现

它们都可用于实现 one loop per thread 模式。 相比 `ACE_TP_Reactor`：

* `ACE_TP_Reactor` 继承于 `ACE_Select_Reactor` ，**加锁** 保证多线程竞争同一 Reactor 的安全。并行可扩展能力受限，且最大支持 1024 个描述符。
* 基于 `ACE_Dev_Poll_Reactor` 的 one loop er thread，每个线程拥有独立的 Reactor，线程之间不存在竞争。充分发挥并行能力，且最大支持几十万甚至百万个描述符。

以下以 `ACE_Select_Reactor` 为例浅析实现的关键代码。更完备示例代码请参看我的 [ace\_echod](https://github.com/likema/ace_echod)

## 实现浅析

### 事件循环函数

`event_loop` 将作为线程函数运行于独立线程中，它与 ACE 教科书基本一样：

```cpp
ACE_THR_FUNC_RETURN event_loop(ACE_Reactor* reactor)
{
    reactor->owner(ACE_OS::thr_self()); // 将reactor “绑定” 本线程。
    while (!reactor->reactor_event_loop_done()) {
        reactor->run_reactor_event_loop();
    }

    return 0;
}
```

### 事件循环管理器

```cpp
class Event_Loop_Manager {
public:
    // 为简化，忽略 ctor 、dtor 和 close

    bool open(unsigned threads)
    {
        // 创建Reactor数组。
        reactors_.reset(new ACE_Reactor*[threads]);
        memset(reactors_.get(), 0, sizeof(ACE_Reactor*) * threads);

        // 创建线程ID数组
        tids_.reset(new ACE_thread_t[threads]);
        memset(tids_.get(), 0, sizeof(ACE_thread_t*) * threads);

        // 创建事件循环
        threads_ = threads;
        for (unsigned i = 0; i < threads_; ++i) {
            reactors_[i] = new ACE_Reactor(new ACE_Select_Reactor, 1), false);
            ACE_Thread_Manager::instance()->spawn(
                    (ACE_THR_FUNC) event_loop, reactors[i],
                    THR_NEW_LWP | THR_JOINABLE | THR_INHERIT_SCHED, &tid);
        }

        current_ = 0;
        return true;
    }

    /// 以简单循环方式返回 Reactor
    ACE_Reactor* reactor()
    {
        ACE_Reactor* r = reactors_[current_];
        current_ = (current_ + 1) % threads_;
        return r;
    }
protected:
    ACE_Auto_Basic_Array_Ptr<ACE_Reactor*> reactors_;
    ACE_Auto_Basic_Array_Ptr<ACE_thread_t> tids_;
    unsigned threads_;
    unsigned current_; // 当前分配 Reactor 索引
};
```

### Echo Acceptor

据 ACE 的 Acceptor 模式, `Echo_Handler` 继承于 `ACE_Svc_Handler<ACE_SOCK_STREAM, ACE_NULL_SYNCH>`，它与 ACE 教科书实现相似。

* `Echo_Acceptor` 注册默认 Reactor ，其事件循环运行于主线程。
* 创建 `Echo_Handler` 对象时，事件循环管理器以简单循环方式分配 Reactor 给新对象。

```cpp
class Echo_Acceptor: public ACE_Acceptor<Echo_Handler, ACE_SOCK_ACCEPTOR> {
public:
    int open(const addr_type& local_addr, int flags, int use_select, int reuse_addr)
    {
        flags_ = flags;
        use_select_ = use_select;
        reuse_addr_ = reuse_addr;
        peer_acceptor_addr_ = local_addr;

        if (peer_acceptor_.open(local_addr, reuse_addr) == -1)
            return -1;

        (void) peer_acceptor_.enable(ACE_NONBLOCK);
        ACE_Reactor::instance()->register_handler(
            this, ACE_Event_Handler::ACCEPT_MASK);

        this->reactor(ACE_Reactor::instance());

        // 按逻辑CPU数初始化事件循环（线程）数。
        return loops_.open(reactor_type, ACE_OS::num_processors()) ? 0 : -1;
    }

    virtual int make_svc_handler(handler_type*& sh)
    {
        if (!sh)
            ACE_NEW_RETURN(sh, handler_type, -1);

        // 事件循环管理器给 Echo_Handler 对象分配事件循环。
        sh->reactor(loops_.reactor());
        return 0;
    }

protected:
    Event_Loop_Manager loops_;
};
```

## 存在问题

* 因 `ACE_Reactor::size` 实现[不一致](https://github.com/DOCGroup/ACE_TAO/issues/855#issuecomment-612391823)，除 `ACE_Dev_Poll_Reactor` 返回当前注册的 handler 数量，其它实现返回handler表的容量。导致无法直接通过注册 handler 的数量，实现 [Round-Robin](https://en.wikipedia.org/wiki/Round-robin_scheduling)，即均衡分配 handler
* 因 `ACE_Reactor` 无法获取当前是否等待事件（即空闲）状态，导致无法直接将 handler 分配到空闲事件循环中。

解决上述问题，要么直接修改 ACE 的实现，要么在具体 handler 实现中增加状态或统计变量，两者都需用原子变量来避免加锁。
