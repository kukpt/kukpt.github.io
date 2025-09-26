---
layout: post
title: 物联网端到端调试
date: 2025-09-26 22:37 +0800
---
### Tips
#### Netty
1. 日志输出 (生产环境使用日志会增多)
```java
    ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG));
```

#### Vert.x
1. 日志输出 (生产环境使用日志会增多)
``` java
    new NetServerOptions().setLogActivity(true);
```

#### TcpDump
1. 保存到文件
``` shell
  sudo tcpdump -i any port 1234 -w output_verbose.pcap
```
##### `-i any`
指定要监听的网络接口。使用 any 表示监听所有可用的网络接口（例如 eth0, wlan0, lo 等）。如果想指定特定接口，可以将其替换为接口名称（如：-i eth0）。

#### Wireshark
##### `搜索报文HEX内容`
```
  tcp contains 2A:0F:FE
```
#### ImHex
[ImHex](https://github.com/WerWolv/ImHex) 十六进制编辑器，进行逆向协议也很好用
