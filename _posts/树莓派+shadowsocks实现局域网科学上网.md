---
title: 树莓派+shadowsocks实现局域网科学上网
author: shynan
date: 2022-04-01 18:00:00 +0800
categories: [Blogging, Tutorial]
tags: [树莓派 shadowsocks dnsmasq]
pin: true
---
## 准备工作
树莓派的系统烧录就不在这里介绍了，网上一大堆资料需要的可以去找。这里假设你已经有了一个烧录好系统的树莓派。我这里使用的是树莓派4B，内存大小是2G。想要实现的目的是所有连上家里路由器的设备都可以无感访问内外网。网络结构如下图所示：
![图1](https://gitee.com/shynan/tuchuang/raw/master/img/20220401170154.png)
## 安装shadowsocks-libev
```c
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install shadowsocks-libev
```
在/etc/shadowsocks-libev下创建shadowsocks的配置文件config.json
```c
sudo vim /etc/shadowsocks-libev/config.json
```
填入一下内容:
```c
{
    "server":"xxx.xxx.xxx.xxx",
    "local_address":"0.0.0.0",
    "mode":"tcp_and_udp",
    "server_port":8888,
    "local_port":1080,
    "password":"*****",
    "timeout":60,
    "method":"chacha20-ietf"
}
```
* "xxx.xxx.xxx.xxx"替换为服务器的ip,
* server_port段填ss的服务器端口，如8888
* password段和method分别对应密码和加密方式
### 创建ss服务并允许自启动

```c
sudo vim /etc/systemd/system/ss-redir.service
```
填入以下内容：
```c
Description=ss-redir

Wants=network.target
After=syslog.target network-online.target

[Service]
Type=simple
Environment=GOGC=20
ExecStart=/usr/bin/ss-redir -c /etc/shadowsocks-libev/config.json -b 0.0.0.0 -u -f /var/run/ss-redir.pid
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```
使能ss-redir服务自启动
```c
sudo systemctl daemon-reload
sudo systemctl enable ss-redir.service

/* 启动 ss-redir */
sudo systemctl start ss-redir.service
```
## 使用dnsmasq和chinadns防止DNS污染
* 安装dnsmasq
```c
sudo apt-get install dnsmasq
```
* 修改dnsmasq配置文件
```
sudo vim /etc/dnsmasq.conf
```
最后追加如下内容
```c
no-resolv
server=127.0.0.1#5354
```

* dnsmasq自启动服务
```c
sudo systemctl enable dnsmasq.service
sudo systemctl start dnsmasq.service
```
* 安装chinadns
```c
sudo wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz
sudo tar -xvf chinadns-1.3.2.tar.gz
cd chinadns-1.3.2
./configure && make
sudo cp ./src/chinadns /usr/local/bin
```
* 更新chnroute.txt列表
chnroute.txt中存放的是国内范围的ip地址，可能不是最新的版本，所以最好更新一下。
```c
sudo curl -0 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chnroute.txt
```
* 创建chinadns自启动服务
```c
sudo vim /etc/systemd/system/chinadns.service
```
输入如下内容
```c
Description=chinadns

Wants=network.target
After=syslog.target network-online.target

[Service]
Type=simple
Environment=GOGC=20
ExecStart=/usr/local/bin/chinadns -c /etc/chnroute.txt -m -p 5354 -s 114.114.114.114,8.8.8.8
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```
使能并启动chinadns
```c
sudo systemctl daemon-reload
sudo systemctl enable chinadns.service
sudo systemctl start chinadns.service
```
* 测试一下dns能否使用
```c
sudo apt-get install dnsutils
sudo dig www.pixiv.net @127.0.0.1 -p 53
```
如果有内容返回，说明dns可以工作了
## 使用ipset和iptables设置规则转发
* 安装ipset
```c
sudo apt-get install ipset
```
* 创建配置脚本
```c
sudo vim /usr/local/bin/ipset_generate.sh
```
填入以下内容
```c
#!/bin/sh

# 删除以下4行注释运行一次，可以更新chnroute.txt列表
#sudo rm /etc/chnroute.txt
#sudo rm /etc/ipset.chnip
#sudo rm /etc/iptables.tproxy
#sudo curl -0 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > /etc/chnroute.txt

if [ -e /etc/ipset.chnip ]; then
ipset restore -f /etc/ipset.chnip
else
ipset -N chnip hash:net
for i in `cat /etc/chnroute.txt`; do echo ipset -A chnip $i >> chnip.sh; done
bash chnip.sh
fi
# 持久化 chnip 表
ipset -S chnip > /etc/ipset.chnip

if [ -e /etc/iptables.tproxy ]; then
iptables-restore < /etc/iptables.tproxy
else
# 新建 mangle/SS-UDP 链，用于透明代理内网 udp 流量
iptables -t mangle -N SS-UDP

# 放行保留地址、环回地址、特殊地址
iptables -t mangle -A SS-UDP -d 0/8 -j RETURN
iptables -t mangle -A SS-UDP -d 127/8 -j RETURN
iptables -t mangle -A SS-UDP -d 10/8 -j RETURN
iptables -t mangle -A SS-UDP -d 169.254/16 -j RETURN
iptables -t mangle -A SS-UDP -d 172.16/12 -j RETURN
iptables -t mangle -A SS-UDP -d 192.168/16 -j RETURN
iptables -t mangle -A SS-UDP -d 224/4 -j RETURN
iptables -t mangle -A SS-UDP -d 240/4 -j RETURN

# 放行发往 ss 服务器的数据包，注意替换为你的服务器IP
iptables -t mangle -A SS-UDP -d xxx.xxx.xxx.xxx -j RETURN

# 放行大陆地址
iptables -t mangle -A SS-UDP -m set --match-set chnip dst -j RETURN

# 重定向 udp 数据包至 5300 监听端口
iptables -t mangle -A SS-UDP -p udp -j TPROXY --tproxy-mark 0x2333/0x2333 --on-ip 127.0.0.1 --on-port 5300

# 内网 udp 数据包流经 SS-UDP 链
iptables -t mangle -A PREROUTING -p udp -s 192.168/16 -j SS-UDP

# 新建 nat/SS-TCP 链，用于透明代理本机/内网 tcp 流量
iptables -t nat -N SS-TCP

# 放行环回地址，保留地址，特殊地址
iptables -t nat -A SS-TCP -d 0/8 -j RETURN
iptables -t nat -A SS-TCP -d 127/8 -j RETURN
iptables -t nat -A SS-TCP -d 10/8 -j RETURN
iptables -t nat -A SS-TCP -d 169.254/16 -j RETURN
iptables -t nat -A SS-TCP -d 172.16/12 -j RETURN
iptables -t nat -A SS-TCP -d 192.168/16 -j RETURN
iptables -t nat -A SS-TCP -d 224/4 -j RETURN
iptables -t nat -A SS-TCP -d 240/4 -j RETURN

# 放行发往 ss 服务器的数据包，注意替换为你的服务器IP
iptables -t nat -A SS-TCP -d xxx.xxx.xxx.xxx -j RETURN

# 放行大陆地址段
iptables -t nat -A SS-TCP -m set --match-set chnip dst -j RETURN

# 重定向 tcp 数据包至 1080 监听端口
iptables -t nat -A SS-TCP -p tcp -j REDIRECT --to-ports 1080

# 本机 tcp 数据包流经 SS-TCP 链
iptables -t nat -A OUTPUT -p tcp -j SS-TCP
# 内网 tcp 数据包流经 SS-TCP 链
iptables -t nat -A PREROUTING -p tcp -s 192.168/16 -j SS-TCP

# 内网数据包源 NAT
iptables -t nat -A POSTROUTING -s 192.168/16 -j MASQUERADE
fi
# 持久化 iptables 规则
iptables-save > /etc/iptables.tproxy
```
其中xxx.xxx.xxx.xxx替换为自己服务器ip
* 创建ipset自启动服务
```c
sudo vim /etc/systemd/system/ipset_iptables.service
```
填入以下内容：
```c
Description=ipset_iptables

Wants=network.target
After=syslog.target network-online.target

[Service]
Type=simple
Environment=GOGC=20
ExecStart=/usr/local/bin/ipset_generate.sh
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```
* 使能并开启ipset_iptables服务
```c
sudo systemctl enable ipset_iptables.service
sudo systemctl start ipset_iptables.service
```
## 修改树莓派ip
```
sudo vim /boot/cmdline.txt
```
添加一条内容:
```c
 ip=192.168.2.4
```

```
sudo route add default gw 192.168.2.1 eth0
```
## 测试效果
1. 树莓派和路由器都用网线连接到光猫
2. 设置路由器联网方式为静态ip方式
3. 配置路由器wan口的网关地址和dns地址，都设为树莓派地址（192.168.2.4）
4. 保存设置后重启路由器
5. 通过路由器连接上网，测试可以访问www.google.com,可以愉快的查资料了！！！