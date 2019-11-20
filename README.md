# HA


### Getting docker's private ip address

```sh
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker-compose ps -q)
```

问题和解决：

宿主机需要开启ip_vs  
sudo modprobe ip_vs  

---------------------------------------------------------------------

keepalived配置完成后vip无法平通，虚拟服务器端口无法访问
2019-03-19 14:32:43 灬紫荆灬 阅读数 1885更多
分类专栏： 故障报错
vip无法ping通
keepalived.conf中vip配置好后，通过ip addr可以看到vip已经顺利挂载，但是无法ping通，并且防火墙都已关闭，原因是keepalived.conf配置中默认vrrp_strict打开了，需要把它注释掉。重启keepalived即可ping通。

映射端口无法访问
vip可ping通后，访问vip映射端口无法访问，直接访问real_server的ip和端口可访问。
解决这个问题需要对lvs相关知识进行初步了解，详见《LVS手册》http://www.linuxidc.com/Linux/2016-03/129233.htm
在keepalived.conf中对virtual_server配置有
lb_kind可以设置为NAT、DR、TUN。这个选项直接关系到你做的 virtual_server和real_server能否进行正确映射。

NAT模式和路由器NAT模式类似，用于访问client和real_server在不同网段实现通信。如果你在一个局域网内做负载均衡选用NAT，那恭喜你，你肯定是无法访问。可以做个NAT模式的测试，需要在keepalived主机上配置双网卡，分别在两个不同网段中，如keepalived主机网卡对client地址为10.0.0.0/24,对real_server的地址为192.168.2.0/24。vip设置为10.0.0.164，real_server为192.168.2.67，可采用下面的keepalived.conf配置

vrrp_instance VI_1 {
state MASTER
interface ens37
virtual_router_id 66
priority 100
advert_int 1
authentication {
    auth_type PASS
    auth_pass 1111
}
virtual_ipaddress {
    10.0.0.164/24
}
}

virtual_server 10.0.0.164 80 {
delay_loop 6
lb_algo rr
lb_kind NAT
protocol TCP
real_server 192.168.2.67 80
   {
         weight 1
}
}
配置正确后在keepalived主机上执行systemctl restart keepalived.service。从client上执行curl 10.0.0.164发现还是无法访问。这是由于real_server在接收到请求包后找不到路由进行数据返回，此时需要将keepalived主机作为网关，在real_server上添加回程路由route add default gw 192.168.2.65。192.168.2.65即为keepalived主机。考虑keepalived主机一般双机，因此此处可以用keepalived主机的虚拟IP。 现在再执行curl 10.0.0.164就可以正常返回。

DR模式是在局域网内最简单的映射模式，原理可参见《LVS手册》。但只在keepalived主机上配置lb_kind DR是无法访问到real_server的，DR模式会将目标地址为虚拟IP地址原封不动的传给real_server。real_server发现这不是我的IP，因此会丢弃掉该包，所以这边得欺骗一下real_server，让他认为这是他的地址。做法很简单，在real_server的lo回环口上添加那个虚拟IP，这样real_server就会认为自己就是VIP这台服务器。切记在lo上设置，不要在真实网卡上设置，道理留给大家思考。

ifconfig lo:0 192.168.2.100 netmask 255.255.255.0 up  
ifconfig lo:0 172.20.0.150  netmask 255.255.255.0 up  

-------------------------------------------------------------
要想把用户的请求调度给后端的RS，是需要经过调度算法来实现，LVS常用以下几种调度算法：
固定调度算法：rr，wrr，dh，sh
动态调度算法：wlc，lc，lblc，lblcr

（1）rr 轮叫调度（Round Robin），这种算法是最简单的，不管RS的后端配置和处理能力，均衡的分发下去

（2）wrr 加权轮叫（Weight Round Robin），比上面的算法多了一个权重的概念，可以给RS设置权重，权重越高，那么分发的请求数越多，权重取值范围0-100

（3）LC最少链接（least connection），这个算法会根据后端的RS的连接数来决定把请求发给谁，比如RS1连接数比RS2连接数少，那么请求优先发给RS1

（4）WLC 加权最少链接（Weighted Least Connecttion）比最少链接算法多了一个权重

（5）Dh 目的地址哈希调度（destination hashing）以目的地址为关键字查找一个静态hash表来获得需要的RS

（6）SH 源地址哈希调度（source hashing）以源地址为关键字查找一个静态hash表来获得需要的RS

（7）lblc 最小连接数调度（least-connection）,IPVS表存储了所有活动的连接。LB会比较将连接请求发送到当前连接最少的RS

（8）Lblcr  带复制的基于本地的最少连接：是LBLC算法的改进

