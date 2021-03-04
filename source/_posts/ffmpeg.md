---
title: FFmpeg
date: 2016-03-31 21:05:06
tags: 
    - media
---

ffmpeg使用
===============================

# 发布rtmp流

```
ffmpeg -loglevel verbose -re -i test.flv -vcodec libx264  -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1 -f flv 'rtmp://10.10.159.119:1935/app1/2'
```


# ffmpeg安装

```
./configure --enable-libmp3lame --enable-libx264 --enable-libfdk-aac --enable-gpl --extra-cflags=-I/usr/local/include --extra-ldflags=-L/usr/local/lib --extra-libs=-ldl --enable-nonfree
```
