### 惊群效应
惊群简单来说就是多个进程或者线程在等待同一个事件，当事件发生时，所有线程和进程都会被内核唤醒。唤醒后通常只有一个进程获得了该事件并进行处理，其他进程发现获取事件失败后又继续进入了等待状态，在一定程度上降低了系统性能。

具体来说惊群通常发生在服务器的监听等待调用上，服务器创建监听socket，后fork多个进程，在每个进程中调用accept或者epoll_wait等待终端的连接。

accept和epoll_wait在历史上都存在惊群效应，但目前都已经解决了（具体内核版本待查）。accept的惊群效应解决的比较早，至少在我测试的2.6.18的内核上没有遇到惊群问题。epoll_wait的惊群问题在这个patch [epoll: add EPOLLEXCLUSIVE flag 21 Jan 2016](https://github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898)已经解决了，通过添加EPOLLEXCLUSIVE标志标识在唤醒时，只唤醒一个等待进程，这个时间比较近没有找对应的内核进行验证（待后续验证）。相比而言，内核在解决accept的惊群时是作为一个问题进行了修复，即无需设置标志，而对epoll_wait则作为添加一个功能选项，这主要是因为accept等待的是一个socket，并且这个socket的连接只能被一个进程处理，内核可以很明确的进行这个预设，因此accept只唤醒一个进程才是更优的选择。而对于epoll_wait，等待的是多个socket上的事件，有连接事件，读写事件等等，这些事件可以同时被一个进程处理，也可以同时被多个进程分别处理，内核不能进行唯一进程处理的假定，因此提供一个设置标志让用户决定。

### nginx如何处理惊群
前面提到内核解决epoll的惊群效应是比较晚的，因此nginx自身解决了该问题（更准确的说是避免了）。其具体思路是：不让多个进程在同一时间监听接受连接的socket，而是让每个进程轮流监听，这样当有连接过来的时候，就只有一个进程在监听那肯定就没有惊群的问题。具体做法是：利用一把进程间锁，每个进程中都尝试获得这把锁，如果获取成功将监听socket加入wait集合中，并设置超时等待连接到来，没有获得所的进程则将监听socket从wait集合去除。这里只是简单讨论nginx在处理惊群问题基本做法，实际其代码还处理了很多细节问题，例如简单的连接的负载均衡、定时事件处理等等。

核心的代码如下
```c
void ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ...
    //这里面会对监听socket处理
    //1、获得锁则加入wait集合
    //2、没有获得则去除
    if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
        return;
    }

    ...

    //设置网络读写事件延迟处理标志，即在释放锁后处理
    if (ngx_accept_mutex_held) {
        flags |= NGX_POST_EVENTS;
    } 

    ...

    //这里面epollwait等待网络事件
    //网络连接事件，放入ngx_posted_accept_events队列
    //网络读写事件，放入ngx_posted_events队列
    (void) ngx_process_events(cycle, timer, flags);

    ...

    //先处理网络连接事件，只有获取到锁，这里才会有连接事件
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    //释放锁，让其他进程也能够拿到
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    //处理网络读写事件
    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```
