# 第三章 无线接入网入侵与防御



## 实验步骤

### Evil Twin基础实验

先确认当前使用的网卡支持AP模式

```bash
# 运行如下所示的命令，若出现 * AP字样，则表示当前网卡支持AP模式
iw list | grep AP

# 运行结果：网卡RT3572L支持AP模式
root@OpenWrt:~# iw list | grep AP
                 * AP
                 * AP/VLAN
                 * AP: 0x00 0x10 0x20 0x30 0x40 0x50 0x60 0x70 0x80 0x90 0xa0 0xb0 0xc0 0xd0 0xe0 0xf0
                 * AP/VLAN: 0x00 0x10 0x20 0x30 0x40 0x50 0x60 0x70 0x80 0x90 0xa0 0xb0 0xc0 0xd0 0xe0 0xf0
                 * AP: 0x00 0x20 0x40 0xa0 0xb0 0xc0 0xd0
                 * AP/VLAN: 0x00 0x20 0x40 0xa0 0xb0 0xc0 0xd0
                 * AP/VLAN
                 * #{ managed, AP, mesh point } <= 8,
        Device supports AP scan.
        Driver supports full state transitions for AP/GO clients
```

```bash
# 安装AP配置管理工具：hostapd 和 简易DNS&DHCP服务器：dnsmasq
opkg update && opkg install -y hostapd dnsmasq

# 创建并编辑 /etc/hostapd/hostapd.conf，内容如下实例所示
cat << EOF > /etc/hostapd/hostapd.conf
ssid=just_a_joke
wpa_passphrase=JokePassword
interface=wlan0
auth_algs=3
channel=6
driver=nl80211
hw_mode=g
logger_stdout=-1
logger_stdout_level=2
max_num_sta=5
rsn_pairwise=CCMP
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
EOF

# 备份/etc/dnsmasq.conf
cp /etc/dnsmasq.conf /etc/dnsmasq.conf.orig

# 使用简化版dnsmasq配置文件内容
cat << EOF > /etc/dnsmasq.conf
log-facility=/var/log/dnsmasq.log
#address=/#/10.10.10.1
#address=/google.com/10.10.10.1
interface=wlan0
dhcp-range=10.10.10.10,10.10.10.250,12h
dhcp-option=3,10.10.10.1
dhcp-option=6,10.10.10.1
#no-resolv
log-queries
EOF

# 启动dnsmasq服务
service dnsmasq start

# 启用无线网卡
ifconfig wlan0 up
ifconfig wlan0 10.10.10.1/24

# 配置防火墙将wlan0的流量转发到有线网卡
iptables -t nat -F
iptables -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

# 开启Linux内核的IPv4数据转发功能
echo '1' > /proc/sys/net/ipv4/ip_forward

# 启动hostapd（调试模式启动）
hostapd -d /etc/hostapd/hostapd.conf

# 用抓包器开始对桥接网卡mitm进行抓包
tcpdump -i wlan0 -w wlan0.pcap
```



###### others

一、

设置无线网卡为hide ESSID

airodump -ng mon1 --beacons  《按tab键高亮》 用手机连接该网络，网络出现