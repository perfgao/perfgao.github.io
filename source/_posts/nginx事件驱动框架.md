---
title: nginx事件驱动框架
banner: /static/image/3603b0058ac888a20293eca66687c64d.png
date: 2017-03-17 18:34:23
categories: nginx
tags:
      - nginx
      - 事件驱动
      - epoll
thumbnail: /static/image/3603b0058ac888a20293eca66687c64d.png
---

本文主要阐述个人对**nginx**的核心机制--事件驱动框架理解,分享自己的见得。也希望帮助相关爱好者更清除的了解nginx。文中如有不妥的地方，欢迎指导更正。

首先先了解下，目前流行的web server中，使用最广的Apache在处理请求时，主要是fork出子进程进行处理，其特征就是一个请求占用一个子进程，请求处理完后进程退出销毁。这种处理架构很明显地随着需要http连接增多对系统的资源占用也随之增加，并发能力不高。而对于nginx是如何做到高性能，高并发的呢--nginx的强大就是使用了事件驱动，也就是IO多路复用。对于nginx的几个worker进程而言，她们关注的是事件。（本文主要探讨的是linux环境下使用epoll模块的IO多路复用）

<!-- more -->

### 网络IO处理
使用epoll机制(本文主要探究epoll，对于freebsd的kqueue不做探究)。对网络请求的socket进行epoll监听，使用epoll_wait（在ET或LT模式下）来获取需要处理的事件。

### 定时器

定时器是由红黑树构成，其中每个节点都是一个时间值。这样每次从树中取出一个最小值的节点与当前系统时间做对比，若是小于当期时间则是超时事件，便会触发相应的超时处理。

### 事件的划分
对于accept成功的socket，之后应该是从客户端（浏览器）读取请求，读取请求的时候，若是请求体比较大，可能会经过几次的触发才能读取完。读取完后，就要与上游服务器建立连接，在tcp建立连接的三次握手过程中，加之复杂的网络环境（有可能建立需要花费较长的时间），nginx不可能阻塞着，这时候nginx会将socket加入到epoll中，并设定一个超时时间在定时器中。一旦在有效时间内上游服务器有回应了，便会触发接下来的继续执行。若是超时了，nginx就会进行连接超时的相关处理（此时若是使用的upstream，便会执行向其他上游服务ip的连接流程）。连接建立成功后，nginx会向上游发送请求。请求发送完后，便是接收上游发来的响应。这些过程中，nginx都不会阻塞着等待期待到来的事件发生，而都是加入到epoll中进行监听，并添加相应的超时事件。


```
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    for ( ;; ) {

        ngx_event_cancel_timers();

    }
}
```

```
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)  
{
    timer = ngx_event_find_timer();

    (void) ngx_process_events(cycle, timer, flags);

    ngx_event_expire_timers();
}
```

```
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    events = epoll_wait(ep, event_list, (int) nevents, timer);

    for (i = 0; i < events; i++) {
        if ((revents & EPOLLIN) && rev->active) {
            rev->handler(rev);
        }

        if ((revents & EPOLLOUT) && wev->active) {
            wev->handler(wev);
        }
    }
}
```

```
void
ngx_event_expire_timers(void)
{

    sentinel = ngx_event_timer_rbtree.sentinel;

    for ( ;; ) {
        root = ngx_event_timer_rbtree.root;
        if (root == sentinel) {
            return;
        }

        node = ngx_rbtree_min(root, sentinel);

        if ((ngx_msec_int_t) (node->key - ngx_current_msec) > 0) {
            return;
        }

        ngx_rbtree_delete(&ngx_event_timer_rbtree, &ev->timer);

        ev->timer_set = 0;
        ev->timedout = 1;
        ev->handler(ev);
    }
}
```

ngx_event_t 结构体中有一些标记位，不同的状态对应不同的标志位
