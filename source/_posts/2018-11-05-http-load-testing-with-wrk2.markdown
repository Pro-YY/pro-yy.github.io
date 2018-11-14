---
layout: post
title: "http load testing with wrk2"
date: 2018-11-05 16:36:47 +0800
comments: true
categories: Network
---

## Overviews

How to measure our server's performance?
With this article, we'll discuss and experiment with HTTP server benchmark.

Let's start to recap some of the key concept related:

- Connection

    The number of simultaneous tcp connections, sometimes refered as `Number of Users` in other benchmark tools.

- Latency

    For HTTP request, it is the same as the `Response Time`, measured by `ms`. And it is tested from clients.
    The latency `percentile`, like p50/p90/p99, is the most common QoS metric.

- Throughput

    For HTTP request, it's also refered as `requests/second` or `RPS` for short. Usually, as the number of connections increases, the system throughput goes down.

So, what does `load testing` really mean?

In brief, it's to determine the maximum throughput (the highest RPS), under specified number of connection, with all response time satisfying the latency target.

Thus, we can remark a server capability like this:
> "Our server instance can achieve 20K RPS under 5K simultaneous connections with latency p99 at less than 200ms."

## What's wrk2
[wrk2](https://github.com/giltene/wrk2) is an HTTP benchmarking cli tool, which is considered better than [ab](https://httpd.apache.org/docs/2.4/programs/ab.html) or [wrk](https://github.com/wg/wrk).
With wrk2, we are able to generate some constant throughput load, and its latency detail is more accurate. As a command-line tool, it's quite convenient and fast.

- -d: duration, test time. Note that it has a 10 second calibration time, so this should be specified no shorter than 20s.
- -t: threads num. Just set it to cpu cores.
- -R: or --rate, expected throughput, the result RPS which is real throughput, will be lower than this value.
- -c: connections num. The Number of connections that will be kept open.

## SUT simple implementation

All servers are simple http-server, which simply response `Hello, world!\n` to clients.

- Rust 1.28.0 (hyper 0.12)
- Go 1.11.1 http module
- Node.js 8.11.4
- Python 3.5.2 asyncio

## Testing Workflow
Our latency target: **The 99 percentile is less than 200ms.** It's a fairly high performance in real world.

Due to the calibration time of wrk2, all the test last for 30~60 seconds.
And since our test machine has 2 cpu-threads, our command is like:
```
./wrk -t2 -c100 -d60 -R 18000 -L http://$HOST:$PORT/
```
We iterate to execute the command, and increase the request rate (-R argument) by 500 on each turn until we find the maximum RPS. The whole workflow can be explained as:
![](/images/http-load-testing-with-wrk2/hb_wf.svg)

Then we go on test for a larger number of connections, until the latency target is no longer satisfied or socket connection errors occur. And move to next server.

## Results Analysis
Now, let's feed our output data to plot program with [matplotlib](https://matplotlib.org/3.0.0/), and finally get the whole picture below:
![](/images/http-load-testing-with-wrk2/http_performance_benchmark.svg)

The plot is fairly clear. Rust beats Go even in such an I/O intensive scenario, which shows the non-blocking sockets version of hyper really makes something great. Node.js is indeed slower than Go, while Python's default asyncio event loop have a rather poor performance.
As a spoiler alert, for the even more connections (i.e. 5K, 10K...), both Rust and Go can still hold very well without any socket connection error, though the response time is longer, and Rust still performed better, while the last two may get down.

## Conclusions
In this post, we managed to benchmark the performance of our web server by using wrk2. And with finite experiment steps, we could determine the server's highest throughput under certain number of connections, which meets the specified latency QoS target.

## References
- https://github.com/giltene/wrk2
- https://www.digitalocean.com/community/tutorials/an-introduction-to-load-testing
- https://blog.loadimpact.com/open-source-load-testing-tool-review
