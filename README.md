# XIAOMI-R4A-V1-OPENWRT
小米R4A V1 OP校园网防检测&宽带叠加

基础条件: 校园网采用Web Protal认证

实现功能: 校园网防检测，宽带叠加(DHCP)

采用OPENWRT官方固件(默认禁用WIFI，请全程连接路由器LAN口)

需要用到WinSCP, PuTTY

# OPENWRT官网选择对应版本和型号进行下载

https://downloads.openwrt.org/

ramips/mt7621

![openwrt官网](https://github.com/user-attachments/assets/f316961f-574f-4051-adf4-346a8d044c7d)

# 刷入固件

PC接入路由器LAN口 进入breed

telnet to breed

利用工具架设虚拟服务器，上传.bin文件

wget htts://*****/x.bin

输出信息

#length A/B (*MB) 
  
  Saving to address 0x80001000

接着依次输入

flash erase 0x180000 B

flash write 0x180000 0x80001000 B

顺手刷入事前备份的epprom固件(修复5G信号弱问题)

wget htts://*****/x.bin

flash erase 0x50000 0x10000

flash write 0x50000 0x80001000 0x10000

最后boot flash 0x180000

3.配置openwrt环境

  后台地址192.168.1.1

  用户root 无密码

Software-update lists

filter搜索

安装iptables-nft, ip6tables-nft, kmod-macvlan, mwan3, luci-app-mwan3, ua2f需要手动下载(https://github.com/Zxilly/UA2F)

# 设置网口

进入Interfaces删除lan外所有网口

Devices-Add device configuration

![add devices](https://github.com/user-attachments/assets/d3a00e2e-ab1d-4cb5-8962-bdb08f8454d0)

Device name: vwan1

Device type: MAC VLAN

Base device: WAN

Mode: Private

MAC:手动设置(否则每次重启会变)

Interfaces- Add new interfaces

![add interface](https://github.com/user-attachments/assets/fe02901a-0fbc-4e7b-8664-68598b99202d)

Name: vwan1

Protocol: DHCP client

Device: vwan1

Advanced Settings-Use gateway metric设置跃点

Firewall Setting 选择WAN

网口跃点唯一，不同网口设置不同跃点

按照以上方法可接着创建vwan2，vwan3……..

按自己实际需求来

# 设置叠加(按照自己实际情况设置)

打开Network-MultiWAN Manager

MultiWAN Manager – Interfaces-add 

![mwan3 add interface](https://github.com/user-attachments/assets/4db7b07f-9979-4e8b-8150-f66028b27b4d)

Name: vwan1

Enabled√

Initial state: online

Tracking hostname or IP address: Baidu.com, qq.com

其他不动

MultiWAN Manager – Members-add

![mwan3 add member](https://github.com/user-attachments/assets/f4710976-c02b-4386-bb5b-9e85d7e13f21)

Name: wan1 (不可与MultiWAN Manager – Interfaces重复)

Interface: vwan1

Metric:不懂可不填

Weight:不懂可不填

MultiWAN Manager – Policies

只留下balance，加上刚刚设置好的成员

![MultiWAN Manager - Policies - balanced](https://github.com/user-attachments/assets/3d976d41-2da6-4278-8277-73be8026dab1)

Member used: wan1

# Firewall4 自定义规则

scp to openwrt，修改/etc/config/firewall

在最后一行加上：

config include
         
         option enabled '1'    
         option type 'script'    
         option path '/etc/firewall.user'
         option fw4_compatible '1'

在etc目录下新建文件：firewall.user，这个文件里就是保存自定义规则的，但是需要注意的是，fw4使用的是nfttables命令(安装iptables-nft自动转换)

# NTP&TTL设置

System-Time synchronization

![time-sync](https://github.com/user-attachments/assets/3c7ee0e0-4988-45f5-82ab-962bc3718413)

Enable NTP client √

Provide NTP server√

NTP server candidates: ntp1.aliyun.com, time1.cloud.tencent.com, stdtime.gov.hk, pool.ntp.org
                       
                                          
修改/etc/firewall.user

iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53

iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53

#防时钟偏移检测

iptables -t nat -N ntp_force_local

iptables -t nat -I PREROUTING -p udp --dport 123 -j ntp_force_local

iptables -t nat -A ntp_force_local -d 0.0.0.0/8 -j RETURN

iptables -t nat -A ntp_force_local -d 127.0.0.0/8 -j RETURN

iptables -t nat -A ntp_force_local -d 192.168.0.0/16 -j RETURN

iptables -t nat -A ntp_force_local -s 192.168.0.0/16 -j DNAT --to-destination 192.168.1.1

#通过 iptables 修改 TTL 值

iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64

# 设置UA2F

scp to openwrt，修改/etc/config/autoua2f(开机自动设置ua2f)

#自动配置防火墙（默认开启）（建议开启）

uci set ua2f.firewall.handle_fw=1

#处理内网流量（默认开启），防止在访问内网服务时被检测到。（建议开启）

uci set ua2f.firewall.handle_intranet=1

#开机自启

uci set ua2f.enabled.enabled=1

#处理 443 端口流量（默认关闭），443 端口流量一般为加密得 https 流量，443 端口出现 http 流量的概率较低（建议关闭）

uci set ua2f.firewall.handle_tls=0

#处理微信的 mmtls 流量（默认关闭）（建议关闭）

uci set ua2f.firewall.handle_mmtls=0

可以在网页 http://ua-check.stagoh.com 上测试 UA2F 是否正常工作



# 软件包OPKG更新问题(如遇到)

修改resolv.conf

scp to openwrt，修改/etc/resolv.conf


nameserver 119.29.29.29

nameserver 223.5.5.5

nameserver 8.8.8.8

注意:在 OpenWrt 中修改 resolv.conf 文件后，重启会导致文件恢复到默认状态。这是因为 resolv.conf 文件通常是由 dnsmasq 或其他服务动态生成的

以下永久性操作不建议执行

方法一：使用 chattr 命令

可以使用 chattr 命令将 resolv.conf 文件设置为不可修改

chattr +i /etc/resolv.conf

注意：某些固件可能不支持 chattr 命令。

方法二：取消重绑定保护

在 OpenWrt 后台取消重绑定保护

1.进入 OpenWrt 管理界面。

2.导航到 网络 > DHCP/DNS。

3.取消勾选 重绑定保护 选项。

方法三：创建自定义 resolv.conf 文件

将原来的 resolv.conf 文件移动并创建一个新的自定义文件

ssh to openwrt


mv /etc/resolv.conf /etc/resolv.conf.link
echo "nameserver X.X.X.X" > /etc/resolv.conf


# Final

感谢前人的无私奉献，我也只是把他们的知识综合运用起来，再次感谢他们

Respect！！！

摘自:

解决无限重启和5G问题

https://www.right.com.cn/forum/thread-8298740-1-1.html

关于校园网防检测

https://blog.sunbk201.site

关于UA2F

https://github.com/Zxilly/UA2F

https://learningman.top/archives/304



