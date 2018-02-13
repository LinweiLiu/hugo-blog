---
title: "NAT类型搭建指南"
date: 2018-02-13T12:09:05+08:00
categories: ["NAT"]
draft: true
---

# NAT类型搭建指南

> 1990年代中期，NAT是作为一种解决IPv4地址短缺以避免保留IP地址困难的方案而流行起来的，由于互联网的普及，NAT就成了家庭和小型办公室网络连接上的路由器的一个标准特征，因为对他们来说，申请独立的IP地址的代价要高于所带来的效益，但是，NAT也让主机之间的通信变得复杂，导致了通信效率的降低，为了提高通信效率，需要对NAT进行内网穿透实现P2P通信，在此之前，需要搭建一套完整的NAT环境，本文目的就是为了完成该任务而撰写。

## 预备知识

为了搭建一套完整NAT环境，你需要知道的：

* **NAT类型**
* **Linux网络数据包管理指令iptables**
* **RFC 3489 STUN相关标准协议**

下面将对上述知识进行简要教学，如果已经熟知请跳过。

### NAT类型

根据UDP传输过程中，NAT处理的差别，将NAT分为四种类型：

* **全锥型(Full Cone)**

  只要外部主机获取到全锥型NAT主机的外网IP和Port就能够与处于NAT后的内网主机进行网络通信。

* **限制锥型(Restricted Cone)**

  外部主机获取到该类型主机的IP和Port信息，还需要该类型主机主动向外部主机发送网络包并在超时时间内，外部主机才有权限根据IP和Port信息使用任意端口Port与其建立连接。

* **端口限制锥型(Port Restricted Cone)**
  在限制锥型的条件下，外部主机只能用该类型主机主动发过请求的端口号Port来与其建立连接。

* **对称型(Symmetric)**

  两个外部主机获取到该类型主机的端口号Port不一样，第三个外部主机无法根据前面两个外部主机获取的IP和Port信息与其建立连接，并且每次该类型主机向外发起一次连接使用的IP和Port将会随机分配。

如果想了解更像详细的信息，请参见[RFC 3489](https://www.ietf.org/rfc/rfc3489.txt)



### Linux网络数据包管理指令iptables

Linux使用iptables来管理网络数据包的流向，iptables是以tables、chains和rules来控制网络包的流向，具体流向关系图：

![iptables_03](/media/images/NAT/iptables_03.gif)

iptables就是根据该关系对数据包进行派送和转发，其中nat表就是NAT服务器的规则表格。这里仅仅介绍iptables的机制流程，具体操作和使用教学参见 [鸟哥的Linux私房菜 Linux防火墙与NAT服务器](http://cn.linux.vbird.org/linux_server/0250simple_firewall.php)

### RFC 3489 STUN相关标准协议

STUN是一个协助NAT穿透的服务器，根据RFC 3489协议描述的穿透方案进行设计，主要进行如下几层判断：

![STUN_algorithm](/media/images/NAT/STUN_algorithm.png)

1. **TEST1:** Client发送UDP包到Server1，要求Server1返回Client的IP和Port信息，Client对比本地和Server1返回包的信息，信息一致则无NAT服务进入防火墙判断，否则存在NAT服务进入NAT类型判断
2. **TEST2:** Client发送UDP包到Server1，要求Server1使用Server2把IP和Port信息返回，Client接收到则该NAT为**Full Cone NAT**, 否则进入**TEST1**测试，如果IP和Port信息与之前**TEST1**不一致则为**Symmetric NAT**
3. **TEST3:** Client发送UDP包到Server1，要求Server1使用另外的端口Port把IP和Port信息返回，Client收到则该NAT为**Restricted Cone NAT**， 否则为**Port Restricted Cone NAT**


-------

## NAT服务器搭建

> 在对NAT相关知识有了一定了解后，在Linux下搭建一套完整的NAT服务成为了可能，这里我们使用iptables来进行搭建。
>
> **变量：**
>
> PUBLIC_IP=66.112.215.237   
> PRIVATE_IP=172.17.0.7  
> PUBLIC_INTERFACE=eth0  
>
> 设置前需情况规则：
>
> sudo iptables -F -t nat  
> sudo iptables -F

* 全锥型

```
  sudo iptables -t nat -A POSTROUTING -o $PUBLIC_INTERFACE -j SNAT --to-source $PUBLIC_IP 
  sudo iptables -t nat -A PREROUTING -i $PUBLIC_INTERFACE -j DNAT --to-destination $PRIVATE_IP
```

* 限制锥型

```
  sudo iptables -t nat -A POSTROUTING -o $PUBLIC_INTERFACE -j SNAT --to-source $PUBLIC_IP
  sudo iptables -t nat -A PREROUTING -i $PUBLIC_INTERFACE -j DNAT --to-destination $PRIVATE_IP 
  sudo iptables -A FORWARD -i $PUBLIC_INTERFACE -m state --state ESTABLISHED,RELATED -j ACCEPT
  sudo iptables -A FORWARD   -s 139.162.62.29 -j ACCEPT #139.162.62.29 STUN服务器地址，允许通过
  sudo iptables -A FORWARD   -d $PRIVATE_IP  -m state --state NEW -j DROP
```

* 端口限制锥型
```
sudo iptables -t nat -A POSTROUTING -o $PUBLIC_INTERFACE -j SNAT --to-source $PUBLIC_IP
```

* 对称型
```
sudo iptables -t nat -A POSTROUTING -o $PUBLIC_INTERFACE -j MASQUERADE --random
```
### 类型测试

使用Python编写RFC3489协议STUN逻辑业务进行NAT类型判断，这里在[pystun](https://github.com/jtriley/pystun)开源工具上进行修改，该工具用来检测NAT类型，目前已经停止维护，代码BUG未能及时修复，只能自行根据RFC3489 STUN协议对代码进行审核，这里使用stun.pjsip.org作为STUN服务器，根据抓包工具Wireshark来进行分析该STUN的逻辑，如图：

![stun](/media/images/NAT/stun.jpg)

根据STUN服务器的业务流程，使用iptables对网络包进行筛选从而实现NAT类型的变换，不同的STUN服务器对IP和Port的使用存在差异可能导致类型检测出错，所以请熟悉RFC3489协议根据STUN服务器的流程进行判决NAT类型。