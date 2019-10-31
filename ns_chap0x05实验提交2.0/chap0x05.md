# 基于 Scapy 编写端口扫描器

## 实验目的

- 掌握网络扫描之端口状态探测的基本原理

## 实验环境

- python + [scapy](https://scapy.net/)

- kali虚拟机两台（victim-1和victim-2）

- 实验网络环境拓扑

  ![](\images\tuopu.jpg)

## 实验要求

- 禁止探测互联网上的 IP ，严格遵守网络安全相关法律法规
- 完成以下扫描技术的编程实现
  - [x] TCP connect scan / TCP stealth scan
  - [x] TCP Xmas scan / TCP fin scan / TCP null scan
  - [x] UDP scan
- 上述每种扫描技术的实现测试均需要测试端口状态为：`开放`、`关闭` 和 `过滤` 状态时的程序执行结果
- 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因；
- 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的
- （可选）复刻 `nmap` 的上述扫描技术实现的命令行参数开关

## 实验过程

#### TCP connect scan

* 80端口开启

  ```
  nc -l -p 80
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

* 80端口开启时

  运行以上scapy代码
  
  ![](\images\TCPconnectscanOPEN2.png)
  
  ![](images\TCPconnectscanOPEN1.png)
  
  用namp扫描，结果相符。
  
  ![](images\TCPopen.png)
  
* nc -l -p [port] 开启端口，crtl+C结束即关闭：

  ![](images\TCPconnectscanCLDSED2.png)
  
  我们可以看到只收到了一个RST包，证明端口处于关闭状态。
  
  ![](images\TCPconnectscanCLOSED1.png)
  
* 添加过滤规则，过滤80端口：

  ```
   iptables -A INPUT -p tcp --dport 80 -j DROP
  ```
  
  wireshark抓包结果
  
  ![](images\TCPconnectscanFiltered.png)
  
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

* 开启|过滤80端口：

  nmap扫描端口结果为开放/过滤
  
  ![](\images\TCPfiltered.png)
  
  开启tcpdump抓包后运行以上代码
  
  ![](images\TCPXMASscanOPEN1.png)
  
  查看抓包文件：
  
  ![](images\TCPXMASscanOPEN2.png)
  
  可以看到开放或过滤状态下的端口则无任何响应，符合预期结果。
  
* 关闭80端口：

  nmap端口扫描
  
  ![](images\TCPclose.png)
  
  运行以上代码。可以看到端口关闭则目标端口会响应 RST 报文。
  
  ![](images\TCPXMASscanCLOSED2.png)                
  
  

#### UDP scan

* 开启53端口

  ```
  nc -l -p 53 -u < /etc/passwd
  ```

* scapy扫描

  ```python
  import logging
  logging.getLogger("scapy.runtime").setLevel(logging.ERROR)
  from scapy.all import *
  
  dst_ip = "172.16.111.120"
  src_port = RandShort()
  dst_port= 53
  dst_timeout=10
  
  def udp_scan(dst_ip,dst_port,dst_timeout):
  	udp_scan_resp = sr1(IP(dst=dst_ip)/UDP(dport=dst_port),timeout=dst_timeout)
      if (str(type(udp_scan_resp))=="<type 'NoneType'>"): #no response
  		print("open|flitered")
  	elif (udp_scan_resp.haslayer(UDP)): # response  open
  		print("open")
  	elif(udp_scan_resp.haslayer(ICMP)): # response icmp
  		if(int(udp_scan_resp.getlayer(ICMP).type)==3 and int(udp_scan_resp.getlayer(ICMP).code)==3):#desination unreachable
  			print("closed")		elif(int(udp_scan_resp.getlayer(ICMP).type)==3 and int(udp_scan_resp.getlayer(ICMP).code) in [1,2,9,10,13]):#filter
  			print("closed")
  elif(udp_scan_resp.haslayer(IP) and udp_scan_resp.getlayer(IP).proto==IP_PROTOS.udp):
      print ("Open")
  udp_scan(dst_ip,dst_port,dst_timeout)
  ```

* 开启53端口时

  nmap扫描端口验证，是开启的

  ![](images\UDPopen1.png)

  查看抓包文件可以看到目标主机响应ipv4包，协议为UDP

  ![](images\UDPopen2.png)

* 关闭53端口时

  nmap端口扫描验证，是关闭的

  ![](images\UDPclosed1.png)

  ![](images\UDPclosed2.png)

  打开抓包文件发现没有UDP回复包，在ICMP包中type和code均为3。

* 设置端口为过滤

  和关闭状态下的端口情况相同。

  ![](images\UDPfiltered1.png)

  nmap扫描验证

  ![](images\UDPfiltered2.png)

#### 遇到的问题

* 先前使用命令 nc -l -p 53 -n 打开udp53端口，但nmap扫描一直显示端口关闭，后改为nc -l -p 53 -n < /etc/passwd 后成功。
* 启动虚拟机时查询本机ip地址发现不是以172开头，原来是在启动时虚拟机网卡默认连接eth1而不是eth0 ，需要手动切换。

#### 参考资料

https://blog.csdn.net/jackcily/article/details/83117884 scapy代码参考

课程视频

