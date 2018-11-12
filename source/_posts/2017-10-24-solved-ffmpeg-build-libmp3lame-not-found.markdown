---
layout: post
title: "[solved] FFMPEG: libmp3lame not found"
date: 2017-10-24 15:09:04 +0800
comments: true
categories: FFmpeg
---

### ERROR: libmp3lame >= 3.98.3 not found

This configure error occurs when building ffmpeg from with mp3 library (libmp3lame) enabled.

solution: add **-lm** to **extra-libs**, like this
```
./configure \
--extra-libs="-lpthread -lm"
...  # other configure options
```
Done!


#### HOW
Check *ffbuild/config.log*, which shows dynamic linking error of libmp3lame, which requires the math lib.
```
util.c:(.text+0x96d): undefined reference to `__exp_finite'
util.c:(.text+0xa41): undefined reference to `__pow_finite'      
layer3.c:(.text+0x2250): undefined reference to `sin'
layer3.c:(.text+0x226a): undefined reference to `cos'       
```
Yeah!
