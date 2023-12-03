---
title: Shadowsocks install&config
date: 2017-11-04 22:42:12
layout: post
tags:
- tools
---

Please install Python at first.

## install
`pip install shadowsocks`

## write config file:
(I am in `~/shadowsocks/config.json`)
```
{
 "server":"0.0.0.0",
 "server_port":8388,
 "local_port":1080,
 "password":"shadowsocks",
 "timeout":600,
 "method":"aes-256-cfb"
}
```

## start:
`ssserver -c ~/shadowsocks/config.json`

## start in background:
`ssserver -c ~/config.json -d`
