---
title: nginx 防止惊群
banner: /static/image/435914a6f392e4fd125c40fe0c905200.jpg
date: 2017-03-24 18:43:21
categories: nginx
tags:
     - nginx
     - 惊群
---

惊群
---------
 惊群现象就是多个进程同时等待同一个事件，当有事件发生时，所有进程都被唤醒，同时去响应这个事件，最后只有一个进程能成功执行，而其他进程只能继续重新休眠。nginx 在master/workers模型下，多个worker进程进行epoll_wait，当有新的连接事件发生，会唤醒所有进程的epoll_wait，结果有些进程发现accpte是失败的。这样的话，开销变大，性能也会随之下降。

<!-- more -->

nginx的解决方案
-------
解决这个问题的方案总结下来就是一句话：
**同一时刻只允许有一个worker进程将需要监听的句柄加入到自己的epoll中**。

accpet锁机制
---------
通过使用加锁的机制，来做进程间的协调。如果开启了accept锁，则每个worker进程每次在进入epoll_wait获取事件之前，会进行尝试获取锁。只有获取到锁的worker进程才能将需要监听的句柄加入到自己的epoll中，进而去处理accpet事件。
不过，这里要注意几个逻辑：
* 如果该worker进程获取锁成功：
    * 并且其**上一次**尝试获得锁是失败的（即上一次没有获得到锁），才会将现有的cycle->listening数组中需要监听的句柄加入到自己的epoll中，
    * 要是**上一次**尝试获得锁也是成功的（即上一次该进程得到了锁，这次仍是该进程拥有），则会直接返回。
* 如果该worker进程获取锁失败，
    * 并且其**上一次**尝试获得锁也是失败的，则会直接返回。
    * 如果其**上一次**尝试获得锁是成功的，则会将现有的cycle->listening数组中已经监听的句柄从自己的epoll中移除。

POST事件
-------------
对于得到锁的进程，会携带**NGX_POST_EVENTS**标记属性。这样在epoll_wait后，会把需要处理的事件先加入到队列中，延迟处理，目的就是避免该进程长时间占有锁。
特别说明一下，这里的延迟队列分为两种：
* accept事件的延迟队列
* read/write事件的延迟队列

也就是对应的事件会加到对应的延迟队列中。进程在释放锁之前，会先处理accept事件的延迟队列，处理完后才会释放锁，然后再去处理超时事件和read/write事件的延迟队列，这样既保证的处理accpet的可靠性，也避免了长时间对锁的占用。

未得到锁的进程
-------------
与此同时，未得到锁的进程会在epoll_wait下至少等待ngx_accept_mutex_delay毫秒，因为有可能超时事件的发生时间会短与ngx_accept_mutex_delay，也有可能有普通的读写事件触发。总之，该进程会更新进程时间缓存，处理相关的事件。然后再次尝试锁的获取。

指令 accept_mutex
----------------
* 默认是打开。

与它相关的指令：
* accept_mutex_delay -- 默认 500 毫秒

看下代码逻辑实现
---------------
* ngx_use_accept_mutex: 当开启了accept_mutex，且worker进程数 > 1, ngx_use_accept_mutex会置为1
* ngx_process_events 是一个函数指针。普通网络套接字调用ngx_epoll_process_events函数开始处理，异步文件i/o设置事件的回调方法为ngx_epoll_eventfd_handler。这里主要看下ngx_epoll_process_events函数逻辑。
* 注意：这里仅仅是为了方便阅读，只是展现了大概的代码逻辑，省略了部分代码。

```
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{

    if (ngx_use_accept_mutex) {

        if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
            return;
        }

        if (ngx_accept_mutex_held) {
            flags |= NGX_POST_EVENTS;
        } else {
            if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
            {
                timer = ngx_accept_mutex_delay;
            }
        }
    }

    (void) ngx_process_events(cycle, timer, flags);

    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    if (delta) {
        ngx_event_expire_timers();
    }

    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

```
ngx_int_t
ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
{
    if (ngx_shmtx_trylock(&ngx_accept_mutex)) {

        if (ngx_accept_mutex_held && ngx_accept_events == 0) {
            return NGX_OK;
        }

        if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
            ngx_shmtx_unlock(&ngx_accept_mutex);
            return NGX_ERROR;
        }

        ngx_accept_events = 0;
        ngx_accept_mutex_held = 1;  

        return NGX_OK;
    }

    if (ngx_accept_mutex_held) {
        if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
            return NGX_ERROR;
        }

        ngx_accept_mutex_held = 0;
    }

    return NGX_OK;
}

```

```
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    events = epoll_wait(ep, event_list, (int) nevents, timer);  

    ngx_time_update();

    for (i = 0; i < events; i++) {

        if ((revents & EPOLLIN) && rev->active) {

            if (flags & NGX_POST_EVENTS) {
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;

                ngx_post_event(rev, queue);

            } else {
                rev->handler(rev);
            }

        }

        if ((revents & EPOLLOUT) && wev->active) {

            if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);
            } else {
                wev->handler(wev);
            }

        }
    }
}
```
