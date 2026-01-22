# Linux 局域网调试指南（工控设备）

[![English](https://img.shields.io/badge/Docs-English-blue.svg)](network_linux.md)

本文档专门针对工控设备的局域网调试，介绍 Linux/嵌入式系统中常用的网络诊断命令和故障排查方法。

## 📋 目录

- [局域网连通性测试](#局域网连通性测试)
- [IP 配置与管理](#ip-配置与管理)
- [设备发现与 ARP](#设备发现与-arp)
- [端口连通性测试](#端口连通性测试)
- [局域网故障排查](#局域网故障排查)
- [静态 IP 配置](#静态-ip-配置)
- [防火墙快速配置](#防火墙快速配置)
- [网络抓包分析](#网络抓包分析)

---

## 局域网连通性测试

### ping - 测试设备连通性

测试嵌入式设备与工控设备之间的网络连通性，是故障排查的第一步。

#### 基本用法

```bash
# 测试工控设备连通性（发送 4 个包）
ping -c 4 192.168.1.100

# 持续监控设备在线状态（按 Ctrl+C 停止）
ping 192.168.1.100

# 快速 ping（间隔 0.2 秒）
ping -i 0.2 192.168.1.100

# 设置超时时间（秒）
ping -W 2 192.168.1.100

# 指定数据包大小（字节）
ping -s 1000 192.168.1.100

# 设置 TTL 值
ping -t 64 192.168.1.100

# 指定网络接口
ping -I eth0 192.168.1.100

# 静默模式（只显示统计）
ping -c 4 -q 192.168.1.100
```

#### 输出解读

```
PING 192.168.1.100 (192.168.1.100) 56(84) bytes of data.
64 bytes from 192.168.1.100: icmp_seq=1 ttl=64 time=0.234 ms
64 bytes from 192.168.1.100: icmp_seq=2 ttl=64 time=0.186 ms
64 bytes from 192.168.1.100: icmp_seq=3 ttl=64 time=0.198 ms
64 bytes from 192.168.1.100: icmp_seq=4 ttl=64 time=0.205 ms

--- 192.168.1.100 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 0.186/0.205/0.234/0.025 ms
```

#### 工控设备调试场景

```bash
# 场景1：检查设备是否在线
ping -c 1 -W 1 192.168.1.100 && echo "设备在线" || echo "设备离线"

# 场景2：测试网络稳定性（检查丢包）
ping -c 100 -i 0.2 192.168.1.100 | grep "packet loss"

# 场景3：批量检查多个设备
for i in {1..254}; do
    ping -c 1 -W 1 192.168.1.$i >/dev/null 2>&1 && echo "192.168.1.$i 在线"
done

# 场景4：检查是否存在 IP 冲突
# 先 ping 目标 IP，然后检查 ARP 表
ping -c 1 192.168.1.100
arp -n | grep 192.168.1.100

# 场景5：监控设备连接状态并记录
ping 192.168.1.100 | while read line; do
    echo "$(date): $line" >> device_monitor.log
done
```

---

## IP 配置与管理

### ip / ifconfig - 查看和管理 IP 配置

工控设备通常使用静态 IP 地址，需要确保嵌入式设备与工控设备在同一网段。

#### 使用 ip 命令（推荐）

```bash
# 查看所有网络接口
ip addr show
# 或简写
ip a

# 查看特定接口
ip addr show eth0

# 查看接口统计信息
ip -s link show eth0

# 查看路由表
ip route show

# 查看 ARP 表
ip neigh show
```

#### 使用 ifconfig 命令（传统）

```bash
# 查看所有接口
ifconfig

# 查看特定接口
ifconfig eth0

# 查看简要信息
ifconfig -s
```

#### 输出解读

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
```

| 字段 | 含义 |
|------|------|
| `UP` | 接口已启用 |
| `LOWER_UP` | 物理链路已连接 |
| `inet 192.168.1.10/24` | IPv4 地址和子网掩码 |
| `link/ether` | MAC 地址 |

#### 工控设备调试场景

```bash
# 场景1：检查设备 IP 配置
ip addr show eth0 | grep "inet "
# 确认 IP 地址、子网掩码与设备在同一网段

# 场景2：检查网络接口状态
ip link show eth0
# 确认接口是 UP 状态，且 LOWER_UP（网线已连接）

# 场景3：快速查看 IP 和 MAC
ip -br addr show
# 简洁格式显示所有接口

# 场景4：检查是否有多个网卡
ip link show | grep "^[0-9]"

# 场景5：网段不匹配时的判断
# 设备: 192.168.1.10/24，工控设备: 192.168.0.100
# 两者不在同一网段，需要修改其中一个的 IP 配置
```

#### IP 网段计算

```bash
# 常用网段配置：
# 配置1: IP: 192.168.1.X, 掩码: 255.255.255.0 (/24) (可用: 192.168.1.1-254)
# 配置2: IP: 192.168.0.X, 掩码: 255.255.255.0 (/24) (可用: 192.168.0.1-254)
# 配置3: IP: 10.0.0.X, 掩码: 255.255.255.0 (/24) (可用: 10.0.0.1-254)

# 判断是否同网段：
# IP 地址与子网掩码进行 AND 运算，结果相同则在同一网段
# 示例：
#   设备:   192.168.1.10  & 255.255.255.0 = 192.168.1.0
#   工控:   192.168.1.100 & 255.255.255.0 = 192.168.1.0  ✓ 同网段
#   工控:   192.168.2.100 & 255.255.255.0 = 192.168.2.0  ✗ 不同网段
```

---

## 设备发现与 ARP

### arp - 查看局域网设备

ARP（地址解析协议）缓存记录了 IP 地址与 MAC 地址的对应关系，用于发现局域网内的设备。

#### 基本用法

```bash
# 显示 ARP 缓存表
arp -n
# 或使用 ip 命令
ip neigh show

# 显示特定接口的 ARP 缓存
arp -i eth0

# 显示特定 IP 的 ARP 条目
arp -n | grep 192.168.1.100

# 添加静态 ARP 条目
arp -s 192.168.1.100 AA:BB:CC:DD:EE:FF
# 或使用 ip 命令
ip neigh add 192.168.1.100 lladdr AA:BB:CC:DD:EE:FF dev eth0

# 删除 ARP 条目
arp -d 192.168.1.100
# 或使用 ip 命令
ip neigh del 192.168.1.100 dev eth0

# 刷新所有 ARP 缓存
ip neigh flush all
```

#### ARP 表输出解读

```
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.1.1              ether   11:22:33:44:55:66   C                     eth0
192.168.1.100            ether   AA:BB:CC:DD:EE:FF   C                     eth0
```

| 字段 | 含义 |
|------|------|
| `Address` | 设备的 IP 地址 |
| `HWaddress` | 设备的 MAC 地址 |
| `Flags C` | 完整条目（已确认）|
| `Iface` | 网络接口 |

#### 工控设备发现场景

```bash
# 场景1：发现局域网内的所有设备
# 先批量 ping，激活 ARP 缓存
for i in {1..254}; do
    ping -c 1 -W 1 192.168.1.$i >/dev/null 2>&1 &
done
wait
# 然后查看 ARP 表
arp -n

# 场景2：确认设备 MAC 地址（防止 IP 冲突）
ping -c 1 192.168.1.100
arp -n | grep 192.168.1.100
# 对比设备标签上的 MAC 地址是否一致

# 场景3：检测 IP 冲突
# 如果 ping 通，但 ARP 显示的 MAC 地址与预期不符
# 说明该 IP 已被其他设备占用
arp -n | grep 192.168.1.100

# 场景4：清除错误的 ARP 缓存
# 当设备更换 IP 后，旧的 ARP 缓存可能导致通信失败
ip neigh flush all
ping -c 1 192.168.1.100

# 场景5：导出 ARP 表（记录设备清单）
arp -n > device_list.txt
```

### 设备发现工具

```bash
# 方法1：使用 nmap 扫描（需要安装）
nmap -sn 192.168.1.0/24

# 方法2：使用 arp-scan（需要安装）
arp-scan --interface=eth0 192.168.1.0/24

# 方法3：简单的 shell 脚本扫描
#!/bin/bash
echo "扫描 192.168.1.0/24 网段..."
for i in {1..254}; do
    {
        ping -c 1 -W 1 192.168.1.$i >/dev/null 2>&1
        if [ $? -eq 0 ]; then
            mac=$(arp -n | grep "192.168.1.$i " | awk '{print $3}')
            echo "192.168.1.$i - $mac"
        fi
    } &
done
wait
```

---

## 端口连通性测试

### netstat / ss - 查看网络连接和端口

用于检查工控设备的通信端口和连接状态。

#### 使用 ss 命令（推荐）

```bash
# 查看所有 TCP 连接
ss -tan

# 查看所有 UDP 连接
ss -uan

# 查看监听端口
ss -tln

# 查看与特定设备的连接
ss -tan | grep 192.168.1.100

# 查看特定端口
ss -tan | grep :502      # Modbus TCP
ss -tan | grep :44818    # EtherNet/IP

# 显示进程信息
ss -tanp

# 统计信息
ss -s
```

#### 使用 netstat 命令（传统）

```bash
# 查看所有连接
netstat -an

# 查看 TCP 连接
netstat -tan

# 查看监听端口
netstat -tln

# 显示进程信息
netstat -tanp

# 查看特定端口
netstat -tan | grep :502
```

#### 输出解读

```
State      Recv-Q Send-Q Local Address:Port    Peer Address:Port
ESTAB      0      0      192.168.1.10:54321   192.168.1.100:502
LISTEN     0      128    0.0.0.0:502          0.0.0.0:*
```

| 状态 | 含义 |
|------|------|
| `LISTEN` | 监听中，等待连接 |
| `ESTAB` (ESTABLISHED) | 已建立连接 |
| `TIME-WAIT` | 连接关闭，等待超时 |
| `CLOSE-WAIT` | 等待关闭 |
| `SYN-SENT` | 发送连接请求 |
| `SYN-RECV` | 接收连接请求 |

#### 工控设备调试场景

```bash
# 场景1：检查设备连接状态
ss -tan | grep 192.168.1.100
# 查看与设备的连接是否建立（ESTAB）

# 场景2：检查工控协议端口是否监听
ss -tln | grep :502      # Modbus TCP
ss -tln | grep :44818    # EtherNet/IP
ss -tln | grep :102      # S7 协议

# 场景3：查找占用端口的程序
ss -tanp | grep :502
# 或
netstat -tanp | grep :502

# 场景4：检查所有已建立的连接
ss -tan state established

# 场景5：导出当前连接状态
ss -tan > connections.txt
```

#### 常用工控协议端口参考

| 协议 | 端口 | 说明 |
|------|------|------|
| Modbus TCP | 502 | Modbus TCP 协议 |
| EtherNet/IP | 44818 | 罗克韦尔自动化 |
| PROFINET | 34962/34963/34964 | 西门子工业以太网 |
| S7 | 102 | 西门子 S7 协议 |
| OPC UA | 4840 | OPC 统一架构 |
| IEC 61850 | 102 | 变电站自动化 |
| BACnet/IP | 47808 | 楼宇自控 |
| Fins | 9600 | 欧姆龙协议 |

### nc / telnet - 测试设备端口

测试工控设备的端口连通性。

#### 使用 nc (netcat)

```bash
# 测试端口是否开放
nc -zv 192.168.1.100 502
# -z: 扫描模式（不发送数据）
# -v: 详细输出

# 测试 Modbus TCP 端口
nc -zv 192.168.1.100 502

# 测试 EtherNet/IP 端口
nc -zv 192.168.1.100 44818

# 测试 OPC UA 端口
nc -zv 192.168.1.100 4840

# 测试端口范围
nc -zv 192.168.1.100 100-1000

# 设置超时时间（秒）
nc -zv -w 2 192.168.1.100 502

# 连接到端口（交互模式）
nc 192.168.1.100 502
```

#### 使用 telnet

```bash
# 测试端口连通性
telnet 192.168.1.100 502

# 如果连接成功，会显示：
# Connected to 192.168.1.100.
# Escape character is '^]'.

# 如果连接失败，会显示：
# Unable to connect to remote host: Connection refused

# 退出 telnet：按 Ctrl+]，然后输入 quit
```

#### 工控设备端口测试场景

```bash
# 场景1：批量测试常用工控端口
#!/bin/bash
device="192.168.1.100"
ports=(502 102 4840 44818)

echo "测试设备: $device"
for port in "${ports[@]}"; do
    if nc -zv -w 2 $device $port 2>&1 | grep -q "succeeded"; then
        echo "端口 $port: 开放"
    else
        echo "端口 $port: 关闭"
    fi
done

# 场景2：持续监控端口状态
while true; do
    nc -zv -w 1 192.168.1.100 502 && echo "$(date): 端口 502 正常"
    sleep 5
done
```

---

## 静态 IP 配置

工控设备通常使用静态 IP，需要配置嵌入式设备网络接口与工控设备在同一网段。

### 临时配置（重启后失效）

#### 使用 ip 命令

```bash
# 配置静态 IP（与工控设备通信）
# 假设工控设备 IP: 192.168.1.100，配置嵌入式设备为同网段
sudo ip addr add 192.168.1.10/24 dev eth0

# 启用网络接口
sudo ip link set eth0 up

# 添加默认网关（可选，局域网内可省略）
sudo ip route add default via 192.168.1.1

# 删除 IP 地址
sudo ip addr del 192.168.1.10/24 dev eth0

# 禁用网络接口
sudo ip link set eth0 down
```

#### 使用 ifconfig 命令

```bash
# 配置静态 IP
sudo ifconfig eth0 192.168.1.10 netmask 255.255.255.0 up

# 添加默认网关
sudo route add default gw 192.168.1.1

# 关闭网络接口
sudo ifconfig eth0 down
```

### 永久配置

#### Debian/Ubuntu 系统 (/etc/network/interfaces)

```bash
# 编辑配置文件
sudo nano /etc/network/interfaces

# 添加以下内容：
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    # dns-nameservers 8.8.8.8  # 可选

# 重启网络服务
sudo systemctl restart networking
# 或
sudo /etc/init.d/networking restart
```

#### RHEL/CentOS 系统 (/etc/sysconfig/network-scripts/)

```bash
# 编辑配置文件
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0

# 添加以下内容：
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.10
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
# DNS1=8.8.8.8  # 可选

# 重启网络服务
sudo systemctl restart network
```

#### 使用 systemd-networkd

```bash
# 创建配置文件
sudo nano /etc/systemd/network/eth0.network

# 添加以下内容：
[Match]
Name=eth0

[Network]
Address=192.168.1.10/24
Gateway=192.168.1.1
# DNS=8.8.8.8  # 可选

# 启用并重启服务
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
```

### 常见网段配置示例

```bash
# 示例1：与 192.168.1.100 的设备通信
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip link set eth0 up

# 示例2：与 192.168.0.50 的设备通信
sudo ip addr add 192.168.0.10/24 dev eth0
sudo ip link set eth0 up

# 示例3：与 10.0.0.100 的设备通信
sudo ip addr add 10.0.0.10/24 dev eth0
sudo ip link set eth0 up

# 示例4：配置脚本
#!/bin/bash
INTERFACE="eth0"
IP="192.168.1.10"
NETMASK="24"

sudo ip addr flush dev $INTERFACE
sudo ip addr add $IP/$NETMASK dev $INTERFACE
sudo ip link set $INTERFACE up
echo "已配置 $INTERFACE: $IP/$NETMASK"
ip addr show $INTERFACE
```

### 网络故障快速重置

```bash
# 重启网络服务
sudo systemctl restart networking        # Debian/Ubuntu
sudo systemctl restart network           # RHEL/CentOS
sudo systemctl restart systemd-networkd  # systemd

# 或重启网络接口
sudo ip link set eth0 down
sudo ip link set eth0 up

# 刷新 IP 地址
sudo ip addr flush dev eth0
```

---

## 防火墙快速配置

嵌入式 Linux 系统的防火墙可能阻止与工控设备的通信，需要配置相应规则。

### iptables 防火墙

#### 查看和管理规则

```bash
# 查看当前规则
sudo iptables -L -n -v

# 查看 INPUT 链规则
sudo iptables -L INPUT -n -v

# 查看规则（带行号）
sudo iptables -L INPUT -n --line-numbers
```

#### 允许工控设备通信

```bash
# 允许来自特定 IP 的所有流量（推荐）
sudo iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# 允许整个网段
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# 允许 Modbus TCP（端口 502）
sudo iptables -A INPUT -p tcp --dport 502 -j ACCEPT

# 允许 EtherNet/IP（端口 44818）
sudo iptables -A INPUT -p tcp --dport 44818 -j ACCEPT

# 允许 S7 协议（端口 102）
sudo iptables -A INPUT -p tcp --dport 102 -j ACCEPT

# 允许 OPC UA（端口 4840）
sudo iptables -A INPUT -p tcp --dport 4840 -j ACCEPT
```

#### 删除规则

```bash
# 查看规则行号
sudo iptables -L INPUT -n --line-numbers

# 删除特定行号的规则
sudo iptables -D INPUT 3

# 清空所有规则（临时调试用）
sudo iptables -F
```

#### 保存规则

```bash
# Debian/Ubuntu
sudo iptables-save > /etc/iptables/rules.v4
sudo netfilter-persistent save

# RHEL/CentOS
sudo service iptables save
sudo iptables-save > /etc/sysconfig/iptables
```

### firewalld 防火墙

```bash
# 查看防火墙状态
sudo firewall-cmd --state

# 允许特定端口
sudo firewall-cmd --permanent --add-port=502/tcp    # Modbus TCP
sudo firewall-cmd --permanent --add-port=44818/tcp  # EtherNet/IP
sudo firewall-cmd --permanent --add-port=102/tcp    # S7

# 允许特定 IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept'

# 重载规则
sudo firewall-cmd --reload

# 临时关闭防火墙（调试用）
sudo systemctl stop firewalld
```

### 快速配置脚本

```bash
#!/bin/bash
# 工控设备防火墙配置脚本

echo "配置工控设备防火墙规则..."

# 允许工控网段
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# 允许常用工控协议端口
sudo iptables -A INPUT -p tcp --dport 502 -j ACCEPT    # Modbus TCP
sudo iptables -A INPUT -p tcp --dport 44818 -j ACCEPT  # EtherNet/IP
sudo iptables -A INPUT -p tcp --dport 102 -j ACCEPT    # S7
sudo iptables -A INPUT -p tcp --dport 4840 -j ACCEPT   # OPC UA

# 保存规则
sudo iptables-save > /etc/iptables/rules.v4

echo "防火墙规则配置完成！"
```

---

## 网络抓包分析

### tcpdump - 命令行抓包工具

嵌入式 Linux 系统中最常用的抓包工具。

#### 基本用法

```bash
# 抓取所有接口的包
sudo tcpdump

# 抓取特定接口
sudo tcpdump -i eth0

# 抓取并保存到文件
sudo tcpdump -i eth0 -w capture.pcap

# 抓取特定数量的包
sudo tcpdump -i eth0 -c 100

# 显示详细信息
sudo tcpdump -i eth0 -v
sudo tcpdump -i eth0 -vv
sudo tcpdump -i eth0 -vvv

# 不解析主机名（加快速度）
sudo tcpdump -i eth0 -n

# 不解析主机名和端口名
sudo tcpdump -i eth0 -nn

# 显示 ASCII 内容
sudo tcpdump -i eth0 -A

# 显示十六进制和 ASCII
sudo tcpdump -i eth0 -X
```

#### 过滤器

```bash
# 按 IP 地址过滤
sudo tcpdump -i eth0 host 192.168.1.100
sudo tcpdump -i eth0 src 192.168.1.100
sudo tcpdump -i eth0 dst 192.168.1.100

# 按网段过滤
sudo tcpdump -i eth0 net 192.168.1.0/24

# 按端口过滤
sudo tcpdump -i eth0 port 502
sudo tcpdump -i eth0 port 502 or port 44818

# 按协议过滤
sudo tcpdump -i eth0 tcp
sudo tcpdump -i eth0 udp
sudo tcpdump -i eth0 icmp
sudo tcpdump -i eth0 arp

# 组合过滤
sudo tcpdump -i eth0 'host 192.168.1.100 and port 502'
sudo tcpdump -i eth0 'tcp and dst 192.168.1.100 and port 502'

# 排除过滤
sudo tcpdump -i eth0 'not port 22'
```

#### 工控设备抓包场景

```bash
# 场景1：抓取与设备的所有通信
sudo tcpdump -i eth0 -nn host 192.168.1.100 -w device_comm.pcap

# 场景2：抓取 Modbus TCP 通信
sudo tcpdump -i eth0 -nn port 502 -w modbus.pcap

# 场景3：抓取并实时查看 Modbus 通信
sudo tcpdump -i eth0 -nn -X port 502

# 场景4：抓取特定时间段的数据
sudo timeout 60 tcpdump -i eth0 -nn host 192.168.1.100 -w capture_60s.pcap

# 场景5：读取并分析 pcap 文件
sudo tcpdump -nn -r capture.pcap
sudo tcpdump -nn -r capture.pcap 'port 502'

# 场景6：统计流量
sudo tcpdump -i eth0 -nn host 192.168.1.100 -c 1000 | wc -l
```

### Wireshark（图形界面）

如果嵌入式设备支持图形界面，可以使用 Wireshark。

```bash
# 安装 Wireshark
sudo apt-get install wireshark  # Debian/Ubuntu
sudo yum install wireshark      # RHEL/CentOS

# 或者在嵌入式设备上抓包，传输到 PC 分析
sudo tcpdump -i eth0 -w - | ssh user@pc-ip "wireshark -k -i -"
```

**常用显示过滤器**（在 Wireshark 中使用）：

```
# IP 过滤
ip.addr == 192.168.1.100
ip.src == 192.168.1.100
ip.dst == 192.168.1.100

# 端口过滤
tcp.port == 502
modbus  # Modbus 协议过滤器

# 组合过滤
ip.addr == 192.168.1.100 && tcp.port == 502
```

---

## 局域网故障排查

### 🔍 工控设备连接排查流程

```
┌───────────────────────────────────────────────────────────┐
│          工控设备局域网连接故障排查流程                      │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  1. 检查物理连接                                           │
│     └── 网线、指示灯、设备电源                              │
│              ↓                                            │
│  2. 检查设备 IP 配置                                       │
│     └── ip addr show eth0                                 │
│     └── 确认与工控设备在同一网段                            │
│              ↓                                            │
│  3. 测试设备连通性                                         │
│     └── ping 设备IP                                       │
│              ↓                                            │
│  4. 检查 ARP 表（确认 MAC 地址）                           │
│     └── arp -n | grep 设备IP                              │
│     └── 检测 IP 冲突                                       │
│              ↓                                            │
│  5. 测试设备端口                                          │
│     └── nc -zv 设备IP 端口                                │
│     └── ss -tan | grep 设备IP                             │
│              ↓                                            │
│  6. 检查防火墙                                            │
│     └── sudo iptables -L -n                               │
│     └── 临时关闭测试：sudo iptables -F                     │
│              ↓                                            │
│  7. 使用 tcpdump 抓包分析                                 │
│     └── 查看是否有数据包交互                               │
│              ↓                                            │
│  8. 网络重置（最后手段）                                   │
│     └── 重启网络服务或接口                                 │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 📋 常见问题快速诊断

| 问题现象 | 快速检查命令 | 可能原因 | 解决方案 |
|---------|-------------|---------|---------|
| ping 不通设备 | `ip addr`<br>`ping 设备IP` | 不在同一网段<br>网线故障<br>设备未开机 | 修改 IP 配置<br>检查物理连接<br>检查设备电源 |
| IP 冲突 | `arp -n \| grep 设备IP`<br>`ip neigh flush all` | 多个设备使用同一 IP | 修改设备或其他主机 IP<br>清除 ARP 缓存 |
| 端口连接失败 | `nc -zv 设备IP 端口`<br>`ss -tan` | 设备端口未开放<br>防火墙阻止 | 检查设备配置<br>配置防火墙规则 |
| 网段不匹配 | `ip addr`<br>`ping 设备IP` | 设备与工控设备不在同一网段 | 修改 IP 配置 |
| MAC 地址不匹配 | `arp -n \| grep 设备IP`<br>`ip link show` | IP 被其他设备占用 | 检查网络内所有设备<br>修改 IP 地址 |
| 通信协议错误 | `tcpdump -i eth0 port 502`<br>`ss -tan` | 协议端口错误<br>协议配置错误 | 确认正确的协议端口<br>检查协议参数 |

### 🛠️ 工控设备诊断脚本

保存为 `device_check.sh`：

```bash
#!/bin/bash

# 工控设备局域网连接诊断脚本

DEVICE_IP="192.168.1.100"
DEVICE_PORT="502"
INTERFACE="eth0"

echo "========================================"
echo "工控设备局域网连接诊断"
echo "========================================"
echo ""
echo "目标设备: $DEVICE_IP"
echo "目标端口: $DEVICE_PORT"
echo "网络接口: $INTERFACE"
echo "========================================"
echo ""

echo "[1] 设备网络配置"
echo "----------------------------------------"
ip addr show $INTERFACE | grep "inet "
echo ""

echo "[2] 测试设备连通性"
echo "----------------------------------------"
ping -c 4 -W 2 $DEVICE_IP
echo ""

echo "[3] ARP 表（检查 MAC 地址）"
echo "----------------------------------------"
arp -n | grep $DEVICE_IP
if [ $? -ne 0 ]; then
    echo "未找到 ARP 条目，设备可能不在线"
fi
echo ""

echo "[4] 与设备的连接状态"
echo "----------------------------------------"
ss -tan | grep $DEVICE_IP
if [ $? -ne 0 ]; then
    echo "无活动连接"
fi
echo ""

echo "[5] 端口连通性测试"
echo "----------------------------------------"
nc -zv -w 2 $DEVICE_IP $DEVICE_PORT 2>&1
echo ""

echo "[6] 防火墙规则"
echo "----------------------------------------"
sudo iptables -L INPUT -n | grep $DEVICE_IP
if [ $? -ne 0 ]; then
    echo "未找到针对该设备的防火墙规则"
fi
echo ""

echo "[7] 设备网段扫描"
echo "----------------------------------------"
echo "扫描 $(echo $DEVICE_IP | cut -d'.' -f1-3).0/24 网段..."
for i in {1..254}; do
    ip=$(echo $DEVICE_IP | cut -d'.' -f1-3).$i
    ping -c 1 -W 1 $ip >/dev/null 2>&1 && echo "  $ip 在线" &
done
wait
echo ""

echo "========================================"
echo "诊断完成"
echo "========================================"
echo ""
echo "建议："
echo "1. 如果 ping 不通，检查 IP 配置和物理连接"
echo "2. 如果 ARP 表无记录，检查设备是否开机"
echo "3. 如果端口无连接，检查防火墙规则"
echo "4. 使用 tcpdump 抓包进一步分析"
```

### 快速配置脚本

保存为 `set_ip.sh`：

```bash
#!/bin/bash

# 快速配置网络接口脚本

INTERFACE="eth0"

echo "========================================"
echo "配置网络接口以连接工控设备"
echo "========================================"
echo ""

echo "当前网络接口："
ip link show | grep "^[0-9]" | cut -d: -f2
echo ""

echo "请选择配置方案："
echo "1. 192.168.1.x 网段（设备 IP: 192.168.1.100）"
echo "2. 192.168.0.x 网段（设备 IP: 192.168.0.100）"
echo "3. 10.0.0.x 网段（设备 IP: 10.0.0.100）"
echo "4. 自定义"
echo ""

read -p "请输入选项 (1-4): " choice

case $choice in
    1)
        IP="192.168.1.10"
        NETMASK="24"
        ;;
    2)
        IP="192.168.0.10"
        NETMASK="24"
        ;;
    3)
        IP="10.0.0.10"
        NETMASK="24"
        ;;
    4)
        read -p "请输入 IP 地址: " IP
        read -p "请输入子网掩码 (如 24): " NETMASK
        ;;
    *)
        echo "无效选项"
        exit 1
        ;;
esac

echo ""
echo "配置 $INTERFACE: $IP/$NETMASK"

# 刷新接口
sudo ip addr flush dev $INTERFACE
# 添加 IP 地址
sudo ip addr add $IP/$NETMASK dev $INTERFACE
# 启用接口
sudo ip link set $INTERFACE up

echo ""
echo "当前 IP 配置："
ip addr show $INTERFACE | grep "inet "
echo ""
echo "配置完成！"
```

### 常见故障快速解决方案

#### 问题1：ping 不通设备

```bash
# 步骤1：检查设备和工控设备是否在同一网段
ip addr show eth0

# 步骤2：清除 ARP 缓存后重试
sudo ip neigh flush all
ping -c 4 192.168.1.100

# 步骤3：临时关闭防火墙测试
sudo iptables -F
ping -c 4 192.168.1.100
```

#### 问题2：IP 冲突

```bash
# 步骤1：查看 ARP 表中的 MAC 地址
arp -n | grep 192.168.1.100

# 步骤2：清除 ARP 缓存
sudo ip neigh flush all

# 步骤3：修改 IP 地址
sudo ip addr flush dev eth0
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip link set eth0 up
```

#### 问题3：端口连接失败

```bash
# 步骤1：测试端口是否开放
nc -zv 192.168.1.100 502

# 步骤2：检查防火墙是否阻止
sudo iptables -L INPUT -n | grep 502

# 步骤3：添加防火墙规则
sudo iptables -A INPUT -p tcp --dport 502 -j ACCEPT
```

#### 问题4：网络配置错误

```bash
# 完全重置网络配置
sudo ip addr flush dev eth0
sudo ip link set eth0 down
sudo ip link set eth0 up
sudo systemctl restart networking
```

---

[返回主页](README_zh.md)
