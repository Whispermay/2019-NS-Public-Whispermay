# 基于 Scapy 编写端口扫描器

## 实验目的

- 掌握网络扫描之端口状态探测的基本原理

## 实验环境

- python + [scapy](https://scapy.net/)
- kali虚拟机两台（victim-1和victim-2）
- 实验网络环境拓扑

## 实验要求

- 禁止探测互联网上的 IP ，严格遵守网络安全相关法律法规
- 完成以下扫描技术的编程实现
  - [x] TCP connect scan / TCP stealth scan
  - [x] TCP Xmas scan / TCP fin scan / TCP null scan
  - [ ] UDP scan
- 上述每种扫描技术的实现测试均需要测试端口状态为：`开放`、`关闭` 和 `过滤` 状态时的程序执行结果
- 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因；
- 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的
- （可选）复刻 `nmap` 的上述扫描技术实现的命令行参数开关

## 实验过程

#### TCP connect scan

* 80端口开启

  ```
  nc -l -p [port]
  ```

* tcpdump抓包并保存

  ```
  tcpdump -i eth0 -n -w filename.cap
  ```

* scapy扫描

  ```python
  import logging
  logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
  from scapy.all import *
  
  dst_ip = "172.16.111.120"
  src_ip = "172.16.111.144"
  src_port = RandShort()
  dst_port=80
  tcp_connect_scan_resp = sr1(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="S"),timeout=10) #SYN
  if str(type(tcp_connect_scan_resp))=="<class 'NoneType'>":
  	print("Filtered")
  elif tcp_connect_scan_resp.haslayer(TCP):
  	if tcp_connect_scan_resp.getlayer(TCP).flags == 0x12: #SYN-ACK
  		send_rst = sr(IP(dst=dst_ip)/TCP(sport=src_port,dport=dst_port,flags="AR"),timeout=10)
  		print("Open")
  	elif tcp_connect_scan_resp.getlayer(TCP).flags == 0x14: #RST
  		print("Closed")
  ```

  1. 80端口开启时：

     运行以上scapy代码

     ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPconnectscanOPEN2.png)

     ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPconnectscanOPEN1.png)

  2. nc -l -p [port] 开启端口时，crtl+C结束即关闭：

     ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPconnectscanCLDSED2.png)

     我们可以看到只收到了一个RST包，证明端口处于关闭状态。

     ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPconnectscanCLOSED1.png)

  3. 添加过滤规则，过滤80端口：

     ```
     iptables -A INPUT -p tcp --dport 80 -j DROP
     ```

     ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPconnectscanFiltered.png)

     可以看到只有一个TCP包，没有得到TCP回复。

#### TCP  XMAS scan

* scapy扫描

  ```python
  # -*-coding:utf-8 -*-
  #! /usr/bin/python3
  
  import logging
  logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
  from scapy.all import *
  
  dst_ip = "172.16.111.120"
  src_ip = "172.16.111.144"
  src_port = RandShort()
  dst_port=80
  
  xmas_scan_resp = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="FPU"),timeout=10)
  if str(type(xmas_scan_resp))=="<type 'NoneType'>":
  	print("Open|Filtered")
  elif xmas_scan_resp.haslayer(TCP):
  	if xmas_scan_resp.getlayer(TCP).flags == 0x14:
  		print("Closed")
  	elif xmas_scan_resp.haslayer(ICMP):
  		if int(xmas_scan_resp.getlayer(ICMP).type)==3 and int(xmas_scan_resp.getlayer(ICMP).code) in [1,2,3,9,10,13]:
  			print("Filtered")
  ```

  1. 开启|过滤80端口：

  ​                开启tcpdump抓包后运行以上代码：

  ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPXMASscanOPEN1.png)

  ​                查看抓包文件：

  ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPXMASscanOPEN2.png)

  ​                可以看到开放或过滤状态下的端口则无任何响应，符合预期结果。

  2. 关闭80端口：

  ​                运行以上代码。如果端口关闭则可以响应 RST 报文，但是抓的包中并没有，目前正在想办法解决。

  ![](https://github.com/CUCCS/2019-NS-Public-Whispermay/blob/ns_chap0x05/ns_chap0x01实验提交1.0/images/TCPXMASscanCLOSED2.png)

