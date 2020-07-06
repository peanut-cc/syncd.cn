# Nmap常见扫描方式流量分析




## 环境说明

扫描者：`manjaro linux `， IP地址：`192.168.31.160`

被扫描者：`centos 7`，IP地址：`192.168.31.175`

分析工具：`wireshark`

nmap 版本：version 7.80



## TCP 知识回顾

这里对TCP的三次握手知识进行简单的回顾，方便后面理解`Nmap`的扫描流量

关于TCP协议相关内容看：http://networksorcery.com/enp/default.htm

![NxilvR.png](https://s1.ax1x.com/2020/07/04/NxilvR.png)



**Source Port**：源端口

**Destination Port**：目的端口

**Sequence Number**：序列号。

**Acknowledgment Number**：确认号

**Control Bits**： 包含一下几种（这几个字段这里要有印象，后续关于nmap的扫描都和这里会有关系）：

| 字段 | 含义                                                         |
| ---- | ------------------------------------------------------------ |
| URG  | 紧急指针是否有效。如果设置1，用于通知接收数据方在处理所有数据包之前处理紧急数据包 |
| ACK  | 确认号是否有效。用于确认主机成功接收数据包。如果**Acknowledgment Number**包含有效的确认号码，则设置ACK标志为。例如tcp三次握手的第二步，发送ACK=1和SYN=1 ，就是告知对方它已经收到初始包 |
| PSH  |                                                              |
| RST  | 连接重置                                                     |
| SYN  | 表示建立连接                                                 |
| FIN  | 表示关闭连接                                                 |



下图是TCP三次握手的过程：



![UiyDCF.png](https://s1.ax1x.com/2020/07/06/UiyDCF.png)





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



在`192.168.31.175` 进行tcpdump 命令进行抓包：`tcpdump -i any -w syn_scan.pcap`,  下面是通过wireshare分数据包分析截图：

![NvOWon.png](https://s1.ax1x.com/2020/07/04/NvOWon.png)

从获取的流量可以很容易看到，这种扫描的的方式是在三次握手的最后一步没有回复正常的ACK,而是发送了RST

![UiIVmD.png](https://s1.ax1x.com/2020/07/06/UiIVmD.png)

nmap 利用客户端回SYN，ACK 的这个数据包其实就已经知道22这个端口是开放的，如果我们扫描一个没有开放的端口：

```bash
➜  sudo nmap -p80 192.168.31.175 
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:08 CST
Nmap scan report for 192.168.31.175
Host is up (0.00034s latency).

PORT   STATE  SERVICE
80/tcp closed http
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.34 seconds

```

同样我们看一下在192.168.31.175 上抓的数据包的情况，被扫描主机直接恢复RST，ACK断开连接，namp从而知道被扫描端口是关闭的，数据包如下：

![UiTro4.png](https://s1.ax1x.com/2020/07/06/UiTro4.png)



## 全连接扫描

nmap也可以进行全连接扫描，也就是完成完整的三次握手，当然这种扫描方式的效率是不如SYN扫描的

`nmap -sT -p端口 目标主机`



## NULL扫描

> 是将一个没有设置任何标志位的数据包发送给TCP端口,在正常的通信中至少要设置一个标志位。
>
> 根据FRC 793的要求,**在端口关闭的情况**下,若收到一个没有设置标志位的数据字段,那么主机应该舍弃这个分段,并发送一个RST数据包,否则不会响应发起扫描的客户端计算机。
>
> 也就是说,如果TCP端口处于关闭则响应一个RST数据包,若处于开放则无相应。但是应该知道理由NULL扫描要求所有的主机都符合RFC 793规定,但是windows系统主机不遵从RFC 793标准,且只要收到没有设置任何标志位的数据包时,不管端口是处于开放还是关闭都响应一个RST数据包。但是基于Unix(*nix,如Linux)遵从RFC 793标准,所以可以用NULL扫描。 经过上面的分析,我们知道NULL可以辨别某台主机运行的操作系统是什么操作系统,是为windows呢?还是*unix/linux?

NULL的扫描命令参数：`nmap -sN`

```shell
➜  sudo nmap -sN -p22 192.168.31.175  
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:30 CST
Nmap scan report for 192.168.31.175
Host is up (0.00022s latency).

PORT   STATE         SERVICE
22/tcp open|filtered ssh
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.67 seconds
➜ sudo nmap -sN -p999 192.168.31.175
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:30 CST
Nmap scan report for 192.168.31.175
Host is up (0.00042s latency).

PORT    STATE  SERVICE
999/tcp closed garcon
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.34 seconds
➜  share scp root@192.168.31.175:/root/null.pcap ./     
root@192.168.31.175's password: 
null.pcap                             100% 5188    10.4MB/s   00:00 
```

​		分别扫描了一个开放的端口22 和未开放的端口999，下面的数据包中，可以看到，发送的第一个TCP包中**Control Bits**的所有Flags都没有设置，而开发的22端口没有收到任何回复，而关闭的端口，收到了一个RST数据包，nmap也是通过这种方式来判断端口是否开放。当然这种判断对于windows是不准确的了。

​		null的扫描流量特征还是非常明显的，很容易通过流量分析知道正在被进行null扫描。

![UibRln.png](https://s1.ax1x.com/2020/07/06/UibRln.png)



## FIN扫描

> FIN 原理：
>
> 与NULL有点类似,只是FIN为指示TCP会话结束,在FIN扫描中一个设置了FIN位的数据包被发送后,若响应RST数据包,则表示端口关闭,没有响应则表示开放。此类扫描同样不能准确判断windows系统上端口开放情况。

FIN扫描命令参数：`nmap -sF`

```shell
➜ sudo nmap -sF -p22 192.168.31.175       
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:46 CST
Nmap scan report for 192.168.31.175
Host is up (0.00014s latency).

PORT   STATE         SERVICE
22/tcp open|filtered ssh
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.61 seconds
➜ sudo nmap -sF -p999 192.168.31.175
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:46 CST
Nmap scan report for 192.168.31.175
Host is up (0.00028s latency).

PORT    STATE  SERVICE
999/tcp closed garcon
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.36 seconds
```

对比null扫描，就可以看出来，FIN扫描是发送一个设置FIN的数据包，同样的如果是linux系统，收到这个如果没有收到RST则认为端口是开放的，收到RST则表示端口没有开发，windows依然无法判断。

![UiOBex.png](https://s1.ax1x.com/2020/07/06/UiOBex.png)



## XMAS-TREE扫描

> 扫描原理：
>
> XMAS扫描原理和NULL扫描的类似,将TCP数据包中的URG、PSH、FIN标志位置1后发送给目标主机。在目标端口开放的情况下,目标主机将不返回任何信息

和NULL扫描正好相反，XMAS扫描会把所有的标志为都设置

XMAS-TREE扫描命令参数：`nmap -sX`

```sh
➜ sudo nmap -sX -p22 192.168.31.175
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:53 CST
Nmap scan report for 192.168.31.175
Host is up (0.00028s latency).

PORT   STATE         SERVICE
22/tcp open|filtered ssh
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.55 seconds
➜ sudo nmap -sX -p999 192.168.31.175
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-06 22:54 CST
Nmap scan report for 192.168.31.175
Host is up (0.00021s latency).

PORT    STATE  SERVICE
999/tcp closed garcon
MAC Address: 4C:1D:96:FC:4D:E2 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.34 seconds
```



还是扫描开放的22端口和未开放的999端口，可以看到Xmas扫描会发送一个FIN，PSH，URG被设置的数据包，同样如果是linux系统可以根据是否收到RST包进行判断端口是否开放，windows无法进行判断



![Uix7bn.png](https://s1.ax1x.com/2020/07/06/Uix7bn.png)





## 延伸阅读

- http://www.ruanyifeng.com/blog/2017/06/tcp-protocol.html
- http://networksorcery.com/enp/default.htm






