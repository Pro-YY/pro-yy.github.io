---
layout: post
title: "Installing Memcache with PHP"
date: 2015-07-16 20:34:27 +0800
comments: true
categories: memcache
---

## install memcached

  OS: Ubuntu-14.04 server

  PHP: 5.6.8 source build

### install memcached with apt-get
```
sudo apt-get install memcached
```

### verification
```
echo "stats settings" | nc localhost 11211
```

## install memcache php extension

### download source from pecl

```
wget http://pecl.php.net/get/memcache-2.2.7.tgz
```

### install with pecl
```
/path/to/php/bin/pecl install /path/to/memcache-2.2.7.tgz
```

There exsists the *lib/php/extensions/no-debug-non-zts-XXXXXXX/memcache.so*

### update the *php.ini*
add "extension=memcache.so" to php.ini

### reload php-fpm server
```
kill -USR2 $(cat /path/to/php/var/run/php-fpm.pid) # no need sudo
```
Check the php.info page, make sure that memcache section exsists
