---
layout: post
title: "Shadowsocks Tutorial"
date: 2015-07-29 19:37:13 +0800
comments: true
categories:
---

##Overview

### socks5
Socket Secure (SOCKS) is an Internet protocol that routes network packets between a client and server through a proxy server. SOCKS5 additionally provides authentication so only authorized users may access a server.

Practically, a SOCKS server proxies TCP connections to an arbitrary IP address, and provides a means for UDP packets to be forwarded.SOCKS operates at a lower level than HTTP proxying: SOCKS uses a handshake protocol to inform the proxy software about the connection that the client is trying to make, and then acts as transparently as possible. SOCKS proxies can also forward UDP traffic and work in reverse.

### shadowsocks
[Shadowsocks](https://github.com/shadowsocks/shadowsocks) is a fast tunnel proxy that can help bypass firewalls.

### test environment
Remote VPS OS: Ubuntu-14.04 Server VPS on DigitalOcean

Local Client OS: Ubuntu-14.04 Desktop

##Installation

###install package

```
sudo apt-get install python-pip
sudo pip install shadowsocks
```

###remote server

```
ssserver -k PASSWORD -d start
```
This will run shadowsocks remote server listening at **0.0.0.0:8388**

###local server
```
sslocal -s VPS -k PASSWORD
```
This will run shadowsocks local server listening at **127.0.0.1:1080**

## Client Configuration
### Web Browser

Thanks for proxy setting extension like [FoxyProxy](http://getfoxyproxy.org/) in Firefox, [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif) in Chrome, socks proxy tunnel can be easily configured.

Choose **socks v5** protocal at localhost:1080, and get it work.

### Curl
```
curl -x socks5h://localhost https://www.google.com
```
Note that we specify protocal as **socks5h** instead of socks5, which means we use the specified SOCKS5 proxy, and let the proxy resolve the host name, otherwise it may not resolve hostname. And port 1080 is by default.

### Nodejs Request

[request](https://www.npmjs.com/package/request) utility doesn't support socks proxy by default, so we need [socks5-http-client](https://www.npmjs.com/package/socks5-http-client) and [socks5-https-client](https://www.npmjs.com/package/socks5-https-client) to create agent.

{% codeblock lang:javascript socks-proxy.js %}
var request = require('request');
var url = require('url');
var socks5HttpAgent = require('socks5-http-client/lib/Agent');
var socks5HttpsAgent = require('socks5-https-client/lib/Agent');

var urlString = process.argv[2];
var url = url.parse(urlString);
var agentClass = (url.protocol === 'https:') ? socks5HttpsAgent : socks5HttpAgent;
var agentOptions = {
  socksHost: 'localhost', // defaults to localhost
  socksPort: 1080         // defaults to 1080
};
var options = {
  uri: url,
  agentClass: agentClass,
  agentOptions: agentOptions,
  method: 'GET'
};
request(options, function(err, res, body) {
  if (err) console.log(err.message);
  else {
    console.log(res.statusCode);
    console.log(body);
  }
});
{% endcodeblock %}

Finally, test it.
```
node socks-proxy.js http://www.google.com
node socks-proxy.js https://www.google.com
```
