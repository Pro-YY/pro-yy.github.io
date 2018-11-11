---
layout: post
title: "worker pool with eventfd"
date: 2018-11-09 22:41:08 +0800
comments: true
categories: 
---

## Linux Eventfd Overview

## Implementation
Our per-thread data structure is fairly simple, only contains 3 fields: thread_id, rank (thread index) and the epoll file descriptor, which is created by `main` function.
```c
typedef struct thread_info {
    pthread_t thread_id;
    int rank;
    int epfd;
} thread_info_t;
```

consumer thread routine
```c
static void *consumer_routine(void *data) {
    struct thread_info *c = (struct thread_info *)data;
    struct epoll_event *events;
    int epfd = c->epfd;
    int nfds = -1;
    int i = -1;
    int ret = -1;
    uint64_t v;
    int num_done = 0;

    events = calloc(MAX_EVENTS_SIZE, sizeof(struct epoll_event));
    if (events == NULL) exit_error("calloc epoll events\n");

    for (;;) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS_SIZE, 1000);
        for (i = 0; i < nfds; i++) {
            if (events[i].events & EPOLLIN) {
                log_debug("[consumer-%d] got event from fd-%d",
                        c->rank, events[i].data.fd);
                ret = read(events[i].data.fd, &v, sizeof(v));
                if (ret < 0) {
                    log_error("[consumer-%d] failed to read eventfd", c->rank);
                    continue;
                }
                do_task();
                log_debug("[consumer-%d] tasks done: %d", c->rank, ++num_done);
                close(events[i].data.fd);
            }
        }
    }
}
```
As we can see, the worker thread get the notification by simply polling `epoll-wait()` the epoll-added fd list, and `read()` the eventfd to consume it,  then `close()` to clean it.
And we can do anything sequential within the `do_task`, although it now does nothing.

In short: poll -> read -> close.

producer thread routine
```c
static void *producer_routine(void *data) {
    struct thread_info *p = (struct thread_info *)data;
    struct epoll_event event;
    int epfd = p->epfd;
    int efd = -1;
    int ret = -1;
    int interval = 1;

    log_debug("[producer-%d] issues 1 task per %d second", p->rank, interval);
    while (1) {
        efd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
        if (efd == -1) exit_error("eventfd create: %s", strerror(errno));
        event.data.fd = efd;
        event.events = EPOLLIN | EPOLLET;
        ret = epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &event);
        if (ret != 0) exit_error("epoll_ctl");
        ret = write(efd, &(uint64_t){1}, sizeof(uint64_t));
        if (ret != 8) log_error("[producer-%d] failed to write eventfd", p->rank);
        sleep(interval);
    }
}
```
In producer routine, after creating `eventfd`, we register the event with epoll object by `epoll_ctl()`. Note that the event is set for write (EPOLLIN) and Edge-Triggerd (EPOLLET).
For notification, what we need to do is just write `0x1` (any value you want) to eventfd.

In short: create -> register -> write.

*Source code repository*: [eventfd_examples](https://github.com/Pro-YY/eventfd_examples/)

## Output & Analysis
The expected output is clear as:
![](/images/worker-pool-with-eventfd/eventfd_worker_execution.gif)

You can adjust threads num to inspect the detail, and there's a plethora of fun with it.


But now, let's try something hard. We'll `smoke test` our worker by generate a heavy instant load, instead of the former regular one. And we tweak the producer/consumer thread to 1, and watching the performance.
```c
static void *producer_routine_spike(void *data) {
    struct thread_info *p = (struct thread_info *)data;
    struct epoll_event event;
    int epfd = p->epfd;
    int efd = -1;
    int ret = -1;
    int num_task = 1000000;

    log_debug("[producer-%d] will issue %d tasks", p->rank, num_task);
    for (int i = 0; i < num_task; i++) {
        efd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
        if (efd == -1) exit_error("eventfd create: %s", strerror(errno));
        event.data.fd = efd;
        event.events = EPOLLIN | EPOLLET;
        ret = epoll_ctl(epfd, EPOLL_CTL_ADD, efd, &event);
        if (ret != 0) exit_error("epoll_ctl");
        ret = write(efd, &(uint64_t){1}, sizeof(uint64_t));
        if (ret != 8) log_error("[producer-%d] failed to write eventfd", p->rank);
    }
    return (void *)0;
}
```

**Over 1 million?** Indeed! By using the `ulimit` command below, we can increase the `open files` limit of the current shell, which is usually 1024 by default.
Note that you need to be root.
```
ulimit -n 1048576
```
Since the info of stdout is so much that we redirect the stdout to file *log*.
![](/images/worker-pool-with-eventfd/eventfd_worker_execution_spike.gif)

With my test VM (S2.Medium4 type on [TencentCloud](https://cloud.tencent.com/), which has only 2 vCPU and 4G memory, it takes less than 6.5 seconds to deal with 1 million concurrent (almost) events. And we've seen the kernel-implemented counters and wait queue are quite effiecient.

## Conclusions
## References
