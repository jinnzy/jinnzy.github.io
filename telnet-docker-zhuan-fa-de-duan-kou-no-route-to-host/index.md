# telnet docker 转发的端口no route to host


两个服务器地址：

```
172.17.5.10
192.168.90.100 

```

从`172.17.5.10` `telnet` `192.168.90.100` 提示`noroute to host`

172.17.5.10抓包，发出去了但是没收到回包

```
[root@vlnx005010 ~]# tcpdump -i eth0 -A   port 80 and  host 192.168.90.100 -vvv
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes


11:20:15.637884 IP (tos 0x10, ttl 64, id 37097, offset 0, flags [DF], proto TCP (6), length 60)
    localhost.50038 > localhost.http: Flags [S], cksum 0xcc56 (incorrect -> 0x5db9), seq 321553758, win 29200, options [mss 1460,sackOK,TS val 973542839 ecr 0,nop,wscale 8], length 0
E..<..@.@......
..Zd.v.P.*.^......r..V.........
:...........


```

192.168.90.100抓包，接受到了包，但是没回包

```
[root@SFADMS dts]# tcpdump -A   port 80 -vvv
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes


10:41:31.312762 IP (tos 0x10, ttl 59, id 33216, offset 0, flags [DF], proto TCP (6), length 60)
    172.17.5.10.7598 > SFADMS.localdomain.http: Flags [S], cksum 0x96df (correct), seq 1563107213, win 29200, options [mss 1460,sackOK,TS val 971218509 ecr 0,nop,wscale 8], length 0
E..<..@.;......
..Zd...P]+........r............
9..M........


```

到这里估计和防火墙规则有关系，突然想起来了昨天修改过docker网段，估计是没更新，执行以下操作后，可以正常通信。

故障时没有看相关错误的信息，直接清空了，下回有时间测试一下补充上。

```
iptables -F
systemctl restart docker 


```

