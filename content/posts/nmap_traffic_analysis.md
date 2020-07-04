---
title: "Nmap常见扫描方式流量分析"
date: 2020-07-04T10:38:33+08:00
draft: true
author: "fan"
subtitle: "安全"
tags: ["安全"]
categories: ["安全"]
toc:
  enable: true
  auto: true
---



## 环境说明

扫描者：`manjaro linux `， IP地址：`192.168.31.160`

被扫描者：`centos 7`，IP地址：`192.168.31.175`

分析工具：`wireshark`



## TCP 知识回顾

这里对TCP的三次握手知识进行简单的回顾，方便后面理解`Nmap`的扫描流量









## SYN 扫描

`Nmap` 的默认扫描方式就是SYN扫描，在`192.168.31.160` 执行如下命令进行扫描：

```bash
➜ sudo nmap -p22 192.168.31.175
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-04 11:45 CST
Nmap scan report for 192.168.31.175
Host is up (0.00044s latency).

PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.38 seconds

```



在`192.168.31.175` 进行tcpdump 命令进行抓包：`tcp -i any -w syn_scan.pcap`,  下面是通过wireshare分数据包分析截图：

![NvOWon.png](https://s1.ax1x.com/2020/07/04/NvOWon.png)













