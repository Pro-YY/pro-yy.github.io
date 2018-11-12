---
layout: post
title: "SSH port forwarding"
date: 2015-08-17 09:29:47 +0800
comments: true
categories: Network SSH
---

## Overview

利用SSH端口转发，可以方便实现各类翻墙打洞需求。本文详细介绍SSH端口转发的三种用法，以及其所适用的场景。

## Environment

- **L0**: localhost behind *NAT*, with lan ip **192.168.0.100**
- **L1**: host within same lan of L0, with lan ip **192.168.0.101**
- **R0**: remote host (*cloud vps*) with private ip **10.0.0.100**
- **R1**: remote host (*cloud vps*) with private ip **10.0.0.101**

L0 can ssh (default port 22) to R0 by its public domain name **r0.example.com** with public key, like this:

{% codeblock %}
ssh user@r0.example.com
{% endcodeblock %}

![](/images/ssh-port-forwarding/0.png)
## Forwarding local host (-L)

### Usage

{% blockquote %}
Specifies that the given port on the local (client) host is to be forwarded to the given host and port on the remote side.
{% endblockquote %}

SSH -L is good for exposing a remote port locally.

增强本地主机的访问远程主机（局域网）能力。

### Example
#### 1. Forward L0 to R0

The mongod has the http web service, listening only on localhost:28017. With the following command:

{% codeblock %}
ssh -NL 28018:localhost:28017 user@r0.example.com
{% endcodeblock %}

Then the remote service on 28017 will be accessed on L0's 28018 port from L0.
![](/images/ssh-port-forwarding/1.png)

#### 2. Forward L0 to R1

Suggest that there's an API server on R1, listening only on port 10030 of lan (10.0.0.0/24).
Because R0 can access R1's private address of the same lan, so the following command will make R1's service of port 10030 accessible from L0's local port 10031.

{% codeblock %}
ssh -NL 10031:10.0.0.101:10030 user@r0.example.com
{% endcodeblock %}

Note, use R1's private ip *10.0.0.101* instead of *localhost*.

![](/images/ssh-port-forwarding/2.png)

#### 3. Forward L1 to R1

Say R0 can access R1 through ssh command: ssh secret-user@10.0.0.101, then the following command

{% codeblock %}
ssh -NL 192.168.0.100:2222:10.0.0.101:22 user@r0.example.com
{% endcodeblock %}

will forward L0's port 2222 to the R1's port 22, and even make L0 listening within the lan. So from L1, we can access to R1 by this command:

{% codeblock %}
ssh -p 2222 secret-user@192.168.0.100
{% endcodeblock %}

![](/images/ssh-port-forwarding/3.png)
Awesome!

## Forwarding remote host (-R)
### Usage
{% blockquote %}
Specifies that the given port on the remote (server) host is to be forwarded to the given host and port on the local side.
{% endblockquote %}

So the SSH -R can be useful when accessing a box hidden behind a NAT.

增强远端主机的访问本地主机（局域网）的能力。

### Example
#### 1. Forward R0 to L0

{% codeblock %}
ssh -NR 2222:localhost:22 user@r0.example.com
{% endcodeblock %}

![](/images/ssh-port-forwarding/4.png)

#### 2. Forward R0 to L1

{% codeblock %}
ssh -NR 2222:192.168.0.101:22 user@r0.example.com
{% endcodeblock %}

![](/images/ssh-port-forwarding/5.png)

#### 3. Forward R1 to L1
Unlike local forwarding, remote forwarding from R1 to L1 is not permitted by default due to security policy,
unless we change *sshd_config* file on **R0**, with the additional config line:
```
GatewayPorts = yes
```
then the ultimate tunnel created, with which we can access L1's port on machine R1. Cool?

## Dynamic forwarding (-D)
### Usage
{% blockquote %}
Specifies a local “dynamic” application-level port forwarding...and ssh will act as a SOCKS server.
{% endblockquote %}

### Example
{% codeblock %}
ssh -ND 1080 user@r0.example.com
{% endcodeblock %}
Then, there exists a SOCKS server on L0, and we can use it as a proxy server.
{% codeblock %}
curl -x socks5h://localhost:1080 https://www.google.com
{% endcodeblock %}
![](/images/ssh-port-forwarding/7.png)
