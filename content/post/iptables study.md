---
title: "Iptables学习与记录"
date: 2018-11-11T16:30:03+08:00
draft: false
description: ""
tags:
- "Linux"
- "Network"
categories: 
- "Network"
---
## iptables学习与记录
1. 列出表中的规则

```
iptables -L
```

等同于 iptables --list 列出指定table的所有规则，如果不使用-t 参数指定对应的表，则默认为filter表， filter表中包括 INPUT、 OUTPUT、FORWARD 三条 chain
常用参数：

```
iptables -vnL -t [表名] --line-numbers
```

表名为 nat、filter 或是 mangle 三张表中的一张，如果不填默认是filter表
-v 代表冗余输出， 带上这个参数能够看到对应rule匹配到的数据包数量 (pkts) 以及字节数(bytes)
-n 代表能够全部以数字的方式输出 ip 地址以及端口号， 如果不加这个参数，0.0.0.0/0 会被默认解析成 anywhere
--line-numbers 能够在打印出规则的时候展示出对应的行号

2. 列出指定表的指定链的规则

```
iptables -t [表名] -vnL [链名]
```

3. 使用iptables-save

```
iptables-save -t [表名]
```

使用 iptables-save 也能够查看规则， 也可以使用此命令将规则导出成一个文件

```
iptables-save > [文件名]
```
