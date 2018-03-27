---
title: nginx的时间管理
banner: /static/image/67f4d95e0df4ac61560df133bc0f0da5.jpg
date: 2017-04-04 22:09:10
categories:
    - nginx
tags:
    - nginx
    - timer
    - 时间管理
---

本文主要以work进程为主要研究对象，研究其在epoll模式下的时间管理逻辑。

时间缓存
-------
nginx 为尽量减少性能损耗，对时间的管理，采用了自己对时间进行缓存，以减少系统调用次数(时间更新使用gettimeofday函数)。这样做也是因为服务器基本对时间的精确度有个容忍度。

三类事件触发时间更新
--------------------
* 网络IO事件
* 超时事件
* 定时事件

<!-- more -->

关于指令timer_resolution
------------------------
work进程对时间的更新在每次的epool_wait结束。在没有配置timer_resolution时，是通过网络IO事件和超时事件来触发更新。这种情况下时间的更新就与网络IO事件和超时事件息息相关。在某些情况下，比如没有超时事件注册，且没有网络IO事件发生，epoll_wait有会一直等待。想要准确的更新时间，nginx就提供了指令timer_resolution来设置获取时间的周期值。当配置了timer_resolution时，则是通过定时事件来触发时间更新的。

```
    timer_resolution 100ms;
```
即是通过指定参数来设置更新频率。这种情况下，epoll_wait会阻塞等待一直到读写事件发生或则信号中断(注意此时没有超时时间)。<br>
另外，看过一些资料说现在x86_64对gettimeofday函数的调用开销很小，所以在配置nginx时可以根据需要来决定是否配置该指令或设定适当更新时间的间隔。

从未配置timer_resolution谈起
----
nginx在master/workers下，每个进程都会维护自己的时间。这里主要看下work进程的时间管理逻辑。工作进程的主要工作函数在ngx_worker_process_cycle:
```
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    for ( ;; ) {

        ngx_process_events_and_timers(cycle);

    }
}
```
每个子进程在进行一系列初始化工作之后，会进入到处理事件的循环中。
在循环中会调用 ngx_process_events_and_timers;
该函数是工作进程的核心函数，这里主要看时间相关的逻辑。
```
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;  // 宏，值为 -1
        flags = 0;
    } else {

        timer = ngx_event_find_timer(); //获取离现在最近的超时定时器时间
        flags = NGX_UPDATE_TIME;
    }

    (void) ngx_process_events(cycle, timer, flags);

}
```
她会对ngx_timer_resolution进行判断(注意：配置了timer_resolution，则ngx_timer_resolution为配置的时间值，**未配置则ngx_timer_resolution值为0**). <br>
在没有配置情况下，通过ngx_event_find_timer获取最近的超时时间,flag赋值为NGX_UPDATE_TIME。之后传入ngx_process_events中：
```
   (void) ngx_process_events(cycle, timer, flags);
```
它是一个钩子函数。linux下，nginx use epoll下，事件为普通网络IO套接字，则调用为：**ngx_epoll_process_events**;<br>
事件为异步文件i/o，则调用为： **ngx_epoll_eventfd_handler**

这里主要看下在ngx_epoll_process_events：
```
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    events = epoll_wait(ep, event_list, (int) nevents, timer);

    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {
        ngx_time_update();
    }

    for (i = 0; i < events; i++) {
    }
}
```
在ngx_epoll_process_events中会调用epoll_wait. 在没有设置 ngx_timer_resolution情况下，则传入的timer为获取的最近的超时时间，这样在epoll_wait即使没有读写事件发生，也会触发超时。不管是读写事件还是超时事件，都会因为传入的flag为NGX_UPDATE_TIME，而进行时间的更新。

配置timer_resolution下的时间管理
-----------------------------
这种情况下的时间更新逻辑，要从子进程创建开始：
nginx在创建子进程时，每个子进程初始化时会调用：
**ngx_event_process_init()**;
```
static ngx_int_t
ngx_event_process_init(ngx_cycle_t *cycle)
{

    if (ngx_timer_resolution && !(ngx_event_flags & NGX_USE_TIMER_EVENT)) {
        struct sigaction  sa;
        struct itimerval  itv;

        ngx_memzero(&sa, sizeof(struct sigaction));

        sa.sa_handler = ngx_timer_signal_handler;
        sigemptyset(&sa.sa_mask);

        if (sigaction(SIGALRM, &sa, NULL) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "sigaction(SIGALRM) failed");
            return NGX_ERROR;
        }

        itv.it_interval.tv_sec = ngx_timer_resolution / 1000;
        itv.it_interval.tv_usec = (ngx_timer_resolution % 1000) * 1000;
        itv.it_value.tv_sec = ngx_timer_resolution / 1000;
        itv.it_value.tv_usec = (ngx_timer_resolution % 1000 ) * 1000;

        if (setitimer(ITIMER_REAL, &itv, NULL) == -1) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, ngx_errno,
                          "setitimer() failed");
        }
    }

}
```
   这里就会判断，若是指定了更新时间值，则该进程会设置一个定时器，并设置该进程的信号处理函数为**ngx_timer_signal_handler**:
```
void  
ngx_timer_signal_handler(int signo)  
{  
    ngx_event_timer_alarm = 1;  
}
```
结合以上ngx_worker_process_cycle和ngx_process_events_and_timers逻辑，进入ngx_process_events_and_timers时，会把timer设置为-1,flag为0。即传入ngx_epoll_process_events的timer为-1,flag为0。timer为-1使epoll_wait为永远等待，直到有读写事件发生。当有读写事件发生后，又由于flag为0，且此时若是ngx_event_timer_alarm为0,则不可能进入到时间更新。<br>
而当定时时间到达时，信号处理函数就会设置ngx_event_timer_alarm为1，同时SIGALRM信号会使处于等待的epoll_wait中断，这样便会更新时间了。

最后在说下setitimer()
-------------------
setitimer(), 是linux的API函数。
它本身有两个作用：
* 指定一段时间后才执行某个函数
* 指定每隔一段时间就执行某个函数 <br>

当setitimer到时就会触发SIGALRM信号。而是要完成哪种作用效果，
   则是struct itimerva的设置了。
