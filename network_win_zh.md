# Windows 网络调试指南

[![English](https://img.shields.io/badge/Docs-English-blue.svg)](network_win.md)

本文档专门介绍 Windows 系统中的网络调试命令和故障排查方法。

## 📋 目录

- [网络连通性测试](#网络连通性测试)
- [网络配置查看与管理](#网络配置查看与管理)
- [路由诊断](#路由诊断)
- [DNS 诊断](#dns-诊断)
- [端口与连接管理](#端口与连接管理)
- [高级网络配置 (netsh)](#高级网络配置-netsh)
- [Windows 防火墙](#windows-防火墙)
- [PowerShell 网络命令](#powershell-网络命令)
- [网络抓包工具](#网络抓包工具)
- [网络故障排查流程](#网络故障排查流程)

---

## 网络连通性测试

### ping

Windows 下的 `ping` 命令用于测试网络连通性。

#### 基本用法

```cmd
# 基本 ping 测试（默认发送 4 个包）
ping 192.168.1.1

# 指定发送次数
ping -n 10 192.168.1.1

# 持续 ping（按 Ctrl+C 停止）
ping -t 192.168.1.1

# 指定数据包大小（字节）
ping -l 1000 192.168.1.1

# 不允许分片
ping -f 192.168.1.1

# 设置 TTL 值
ping -i 64 192.168.1.1

# 设置超时时间（毫秒）
ping -w 2000 192.168.1.1

# 指定源地址
ping -S 192.168.1.100 192.168.1.1

# IPv6 ping
ping -6 fe80::1
```

#### 输出解读

```
正在 Ping 192.168.1.1 具有 32 字节的数据:
来自 192.168.1.1 的回复: 字节=32 时间=1ms TTL=64
来自 192.168.1.1 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.1.1 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.1.1 的回复: 字节=32 时间=1ms TTL=64

192.168.1.1 的 Ping 统计信息:
    数据包: 已发送 = 4，已接收 = 4，丢失 = 0 (0% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 0ms，最长 = 1ms，平均 = 0ms
```

#### 常见错误信息

| 错误信息 | 含义 | 排查方向 |
|---------|------|---------|
| `请求超时` | 目标主机未响应 | 检查目标主机状态、防火墙 |
| `无法访问目标主机` | 路由不可达 | 检查网关配置、路由表 |
| `传输失败。常规错误` | 网络适配器问题 | 检查网卡驱动、网线连接 |
| `目标主机无法访问` | 本地网络配置问题 | 检查 IP 配置 |
| `目标端口无法访问` | 端口不通 | 检查防火墙规则 |

---

## 网络配置查看与管理

### ipconfig

`ipconfig` 是 Windows 查看和管理 IP 配置的主要命令。

#### 基本用法

```cmd
# 查看基本网络配置
ipconfig

# 查看详细配置信息
ipconfig /all

# 释放 DHCP 分配的 IP
ipconfig /release

# 重新获取 DHCP 地址
ipconfig /renew

# 刷新 DNS 缓存
ipconfig /flushdns

# 显示 DNS 缓存内容
ipconfig /displaydns

# 释放 IPv6 地址
ipconfig /release6

# 重新获取 IPv6 地址
ipconfig /renew6

# 显示所有适配器信息
ipconfig /allcompartments

# 查看特定适配器
ipconfig | findstr "以太网"
```

#### 输出解读

```
以太网适配器 以太网:

   连接特定的 DNS 后缀 . . . . . . . : localdomain
   本地链接 IPv6 地址. . . . . . . . : fe80::a1b2:c3d4:e5f6:7890%12
   IPv4 地址 . . . . . . . . . . . . : 192.168.1.100
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   默认网关. . . . . . . . . . . . . : 192.168.1.1
```

#### 快速诊断技巧

```cmd
# 检查 IP 配置是否正常
ipconfig | findstr /C:"IPv4" /C:"子网掩码" /C:"默认网关"

# 检查 DNS 服务器
ipconfig /all | findstr /C:"DNS 服务器"

# 查看 MAC 地址
ipconfig /all | findstr /C:"物理地址"

# 重置网络配置（释放并重新获取）
ipconfig /release && ipconfig /renew && ipconfig /flushdns
```

### getmac

查看网络适配器的 MAC 地址。

```cmd
# 查看所有 MAC 地址
getmac

# 详细格式
getmac /v

# 查看远程计算机的 MAC（需要管理员权限）
getmac /s 远程主机名

# 查看特定格式
getmac /fo table
getmac /fo list
getmac /fo csv
```

---

## 路由诊断

### tracert

跟踪数据包到达目标的路由路径。

#### 基本用法

```cmd
# 基本路由跟踪
tracert www.baidu.com

# 不解析 IP 地址（加快速度）
tracert -d 8.8.8.8

# 指定最大跳数
tracert -h 20 8.8.8.8

# 指定超时时间（毫秒）
tracert -w 2000 8.8.8.8

# IPv6 路由跟踪
tracert -6 ipv6.google.com
```

#### 输出解读

```
通过最多 30 个跃点跟踪到 www.baidu.com [220.181.38.150] 的路由:

  1    <1 毫秒   <1 毫秒   <1 毫秒  192.168.1.1
  2     2 毫秒    2 毫秒    2 毫秒  10.0.0.1
  3     *        *        *     请求超时。
  4    15 毫秒   15 毫秒   15 毫秒  220.181.38.150

跟踪完成。
```

| 符号/信息 | 含义 |
|----------|------|
| `*` | 该跳无响应（超时） |
| `请求超时` | 路由器未响应探测包 |
| 三个时间值 | 三次探测的往返时间 |

### pathping

结合 `ping` 和 `tracert`，提供更详细的路由质量分析。

```cmd
# 基本用法
pathping www.baidu.com

# 不解析 IP 地址
pathping -n 8.8.8.8

# 指定最大跳数
pathping -h 20 8.8.8.8

# 指定查询次数（默认 100）
pathping -q 50 8.8.8.8

# 指定查询间隔（毫秒，默认 250）
pathping -p 500 8.8.8.8
```

#### 输出解读

```
正在通过最多 30 个跃点跟踪到 www.baidu.com [220.181.38.150] 的路由:
  0  DESKTOP-PC [192.168.1.100]
  1  192.168.1.1
  2  10.0.0.1
  3  220.181.38.150

正在计算统计信息(需要 125 秒)...
            源到此处   此节点/链接
跃点  RTT    丢失/发送 = Pct  丢失/发送 = Pct  地址
  0                                           DESKTOP-PC [192.168.1.100]
                                0/ 100 =  0%   |
  1    0ms     0/ 100 =  0%     0/ 100 =  0%  192.168.1.1
                                0/ 100 =  0%   |
  2    5ms     0/ 100 =  0%     0/ 100 =  0%  10.0.0.1
                                2/ 100 =  2%   |
  3   20ms     2/ 100 =  2%     0/ 100 =  0%  220.181.38.150
```

### route

查看和管理路由表。

```cmd
# 查看路由表
route print

# 查看 IPv4 路由表
route print -4

# 查看 IPv6 路由表
route print -6

# 添加静态路由
route add 10.0.0.0 mask 255.0.0.0 192.168.1.1

# 删除路由
route delete 10.0.0.0

# 添加永久路由（重启后保留）
route -p add 10.0.0.0 mask 255.0.0.0 192.168.1.1

# 修改路由
route change 10.0.0.0 mask 255.0.0.0 192.168.1.2

# 指定跃点数（metric）
route add 10.0.0.0 mask 255.0.0.0 192.168.1.1 metric 10
```

---

## DNS 诊断

### nslookup

DNS 查询工具。

#### 基本用法

```cmd
# 基本域名解析
nslookup www.baidu.com

# 指定 DNS 服务器
nslookup www.baidu.com 8.8.8.8

# 查询特定记录类型
nslookup -type=A www.baidu.com
nslookup -type=MX baidu.com
nslookup -type=NS baidu.com
nslookup -type=CNAME www.baidu.com
nslookup -type=TXT baidu.com

# 反向 DNS 查询
nslookup 8.8.8.8

# 交互模式
nslookup
> set type=MX
> baidu.com
> exit
```

#### 输出解读

```
服务器:  dns.google
Address:  8.8.8.8

非权威应答:
名称:    www.baidu.com
Addresses:  220.181.38.150
          220.181.38.149
```

| 字段 | 含义 |
|------|------|
| `服务器` | 使用的 DNS 服务器 |
| `非权威应答` | 来自缓存而非权威服务器 |
| `名称` | 域名 |
| `Addresses` | 解析的 IP 地址 |

### 常见 DNS 问题排查

```cmd
# 1. 检查 DNS 服务器配置
ipconfig /all | findstr /C:"DNS 服务器"

# 2. 清除 DNS 缓存
ipconfig /flushdns

# 3. 测试不同 DNS 服务器
nslookup www.baidu.com 8.8.8.8
nslookup www.baidu.com 114.114.114.114
nslookup www.baidu.com 223.5.5.5

# 4. 检查 hosts 文件
notepad C:\Windows\System32\drivers\etc\hosts

# 5. 检查 DNS 客户端服务
sc query Dnscache
net start Dnscache
```

---

## 端口与连接管理

### netstat

查看网络连接、路由表、接口统计等信息。

#### 基本用法

```cmd
# 查看所有连接和监听端口
netstat -a

# 查看监听端口
netstat -an

# 显示可执行文件（需要管理员权限）
netstat -ano

# 显示包含进程 ID 和进程名称
netstat -anob

# 只显示 TCP 连接
netstat -ant

# 只显示 UDP 连接
netstat -anu

# 显示以太网统计
netstat -e

# 显示路由表
netstat -r

# 显示每个协议的统计
netstat -s

# 每隔 N 秒刷新一次
netstat -an 5

# 查看特定端口
netstat -ano | findstr :80
netstat -ano | findstr :3389

# 查看特定进程的连接
netstat -ano | findstr 1234
```

#### 输出解读

```
协议  本地地址          外部地址        状态           PID
TCP   0.0.0.0:80        0.0.0.0:0       LISTENING     4
TCP   192.168.1.100:1234  8.8.8.8:443   ESTABLISHED   5678
```

| 状态 | 含义 |
|------|------|
| `LISTENING` | 监听中，等待连接 |
| `ESTABLISHED` | 已建立连接 |
| `TIME_WAIT` | 连接关闭，等待超时 |
| `CLOSE_WAIT` | 等待关闭 |
| `SYN_SENT` | 发送连接请求 |
| `SYN_RECEIVED` | 接收连接请求 |

#### 实用示例

```cmd
# 查找占用特定端口的进程
netstat -ano | findstr :80
tasklist | findstr "1234"

# 杀掉占用端口的进程
taskkill /F /PID 1234

# 查看建立的连接数
netstat -an | find /c "ESTABLISHED"

# 查看监听的端口
netstat -an | find "LISTENING"

# 查看所有外部连接
netstat -an | find "ESTABLISHED" | find /v "127.0.0.1"
```

### telnet

测试端口连通性（需要启用 Telnet 客户端）。

```cmd
# 启用 Telnet 客户端（需要管理员权限）
dism /online /Enable-Feature /FeatureName:TelnetClient

# 测试端口
telnet 192.168.1.1 80
telnet www.baidu.com 443

# 测试成功会显示空白屏幕或连接信息
# 测试失败会显示"无法打开到主机的连接"
```

### Test-NetConnection (PowerShell)

```powershell
# 测试连接（类似 ping）
Test-NetConnection 192.168.1.1

# 测试特定端口
Test-NetConnection 192.168.1.1 -Port 80

# 显示详细信息
Test-NetConnection www.baidu.com -Port 443 -InformationLevel Detailed

# 路由跟踪
Test-NetConnection www.baidu.com -TraceRoute
```

---

## 高级网络配置 (netsh)

`netsh` 是 Windows 强大的网络配置命令行工具。

### 网络接口配置

```cmd
# 查看所有网络接口
netsh interface show interface

# 查看 IP 配置
netsh interface ip show config

# 设置静态 IP
netsh interface ip set address name="以太网" static 192.168.1.100 255.255.255.0 192.168.1.1

# 设置 DHCP
netsh interface ip set address name="以太网" dhcp

# 设置 DNS 服务器
netsh interface ip set dns name="以太网" static 8.8.8.8
netsh interface ip add dns name="以太网" 8.8.4.4 index=2

# 设置自动获取 DNS
netsh interface ip set dns name="以太网" dhcp

# 启用/禁用网络接口
netsh interface set interface "以太网" enabled
netsh interface set interface "以太网" disabled
```

### 网络配置导出与导入

```cmd
# 导出网络配置
netsh -c interface dump > network_config.txt

# 导入网络配置
netsh -f network_config.txt
```

### 查看网络连接状态

```cmd
# 查看 TCP 连接
netsh interface tcp show global

# 查看所有网络配置
netsh interface ip show config

# 查看 DNS 缓存
netsh interface ip show dnsservers

# 查看 ARP 缓存
netsh interface ip show neighbors
```

### WLAN 配置（无线网络）

```cmd
# 查看 WLAN 信息
netsh wlan show interfaces

# 查看保存的 WLAN 配置文件
netsh wlan show profiles

# 显示特定配置文件的详细信息（包括密码）
netsh wlan show profile name="WiFi名称" key=clear

# 连接到网络
netsh wlan connect name="WiFi名称"

# 断开连接
netsh wlan disconnect

# 删除配置文件
netsh wlan delete profile name="WiFi名称"

# 扫描可用网络
netsh wlan show networks

# 导出 WLAN 配置
netsh wlan export profile folder=C:\WiFiBackup

# 导入 WLAN 配置
netsh wlan add profile filename="C:\WiFiBackup\profile.xml"
```

### 网络重置

```cmd
# 重置 TCP/IP 栈
netsh int ip reset

# 重置 Winsock 目录
netsh winsock reset

# 重置防火墙配置
netsh advfirewall reset

# 重置所有网络设置（需要重启）
netsh int ip reset
netsh winsock reset
ipconfig /flushdns
ipconfig /release
ipconfig /renew
```

---

## Windows 防火墙

### 防火墙基本管理

```cmd
# 查看防火墙状态
netsh advfirewall show allprofiles

# 启用防火墙（所有配置文件）
netsh advfirewall set allprofiles state on

# 禁用防火墙（不推荐）
netsh advfirewall set allprofiles state off

# 启用特定配置文件的防火墙
netsh advfirewall set domainprofile state on
netsh advfirewall set privateprofile state on
netsh advfirewall set publicprofile state on

# 恢复默认防火墙设置
netsh advfirewall reset
```

### 防火墙规则管理

```cmd
# 查看所有防火墙规则
netsh advfirewall firewall show rule name=all

# 查看特定规则
netsh advfirewall firewall show rule name="远程桌面"

# 允许特定端口（入站）
netsh advfirewall firewall add rule name="允许 TCP 80" dir=in action=allow protocol=TCP localport=80

# 允许特定端口（出站）
netsh advfirewall firewall add rule name="允许 TCP 443" dir=out action=allow protocol=TCP localport=443

# 允许特定程序
netsh advfirewall firewall add rule name="允许 MyApp" dir=in action=allow program="C:\Program Files\MyApp\app.exe"

# 允许特定 IP 地址
netsh advfirewall firewall add rule name="允许特定IP" dir=in action=allow remoteip=192.168.1.100

# 阻止特定端口
netsh advfirewall firewall add rule name="阻止 TCP 445" dir=in action=block protocol=TCP localport=445

# 删除规则
netsh advfirewall firewall delete rule name="允许 TCP 80"

# 启用/禁用规则
netsh advfirewall firewall set rule name="远程桌面" new enable=yes
netsh advfirewall firewall set rule name="远程桌面" new enable=no
```

### 防火墙日志

```cmd
# 启用防火墙日志
netsh advfirewall set allprofiles logging filename %systemroot%\system32\LogFiles\Firewall\pfirewall.log
netsh advfirewall set allprofiles logging maxfilesize 4096
netsh advfirewall set allprofiles logging droppedconnections enable
netsh advfirewall set allprofiles logging allowedconnections enable

# 查看日志配置
netsh advfirewall show allprofiles logging

# 查看日志文件
type %systemroot%\system32\LogFiles\Firewall\pfirewall.log
```

---

## PowerShell 网络命令

PowerShell 提供了更强大和灵活的网络管理功能。

### 基本网络诊断

```powershell
# 测试网络连接
Test-Connection -ComputerName 192.168.1.1 -Count 4

# 持续 ping
Test-Connection -ComputerName 192.168.1.1 -Continuous

# 测试端口连通性
Test-NetConnection -ComputerName 192.168.1.1 -Port 80

# 路由跟踪
Test-NetConnection -ComputerName www.baidu.com -TraceRoute

# DNS 解析
Resolve-DnsName www.baidu.com
Resolve-DnsName -Name www.baidu.com -Type A
Resolve-DnsName -Name baidu.com -Type MX

# 清除 DNS 缓存
Clear-DnsClientCache

# 查看 DNS 缓存
Get-DnsClientCache
```

### 网络适配器管理

```powershell
# 查看所有网络适配器
Get-NetAdapter

# 查看启用的适配器
Get-NetAdapter | Where-Object {$_.Status -eq "Up"}

# 查看适配器详细信息
Get-NetAdapter -Name "以太网" | Format-List *

# 启用/禁用适配器
Enable-NetAdapter -Name "以太网"
Disable-NetAdapter -Name "以太网"

# 重启适配器
Restart-NetAdapter -Name "以太网"

# 查看适配器统计
Get-NetAdapterStatistics
```

### IP 配置管理

```powershell
# 查看 IP 配置
Get-NetIPConfiguration

# 查看详细 IP 配置
Get-NetIPConfiguration -Detailed

# 查看特定适配器的 IP
Get-NetIPAddress -InterfaceAlias "以太网"

# 设置静态 IP
New-NetIPAddress -InterfaceAlias "以太网" -IPAddress 192.168.1.100 -PrefixLength 24 -DefaultGateway 192.168.1.1

# 删除 IP 地址
Remove-NetIPAddress -InterfaceAlias "以太网" -IPAddress 192.168.1.100

# 设置 DNS 服务器
Set-DnsClientServerAddress -InterfaceAlias "以太网" -ServerAddresses ("8.8.8.8","8.8.4.4")

# 设置 DHCP
Set-NetIPInterface -InterfaceAlias "以太网" -Dhcp Enabled
```

### 路由管理

```powershell
# 查看路由表
Get-NetRoute

# 查看默认路由
Get-NetRoute -DestinationPrefix "0.0.0.0/0"

# 添加静态路由
New-NetRoute -DestinationPrefix "10.0.0.0/8" -InterfaceAlias "以太网" -NextHop 192.168.1.1

# 删除路由
Remove-NetRoute -DestinationPrefix "10.0.0.0/8"
```

### 防火墙管理

```powershell
# 查看防火墙状态
Get-NetFirewallProfile

# 启用防火墙
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# 禁用防火墙
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# 查看防火墙规则
Get-NetFirewallRule

# 查看特定规则
Get-NetFirewallRule -DisplayName "远程桌面*"

# 创建防火墙规则
New-NetFirewallRule -DisplayName "允许 TCP 80" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

# 删除规则
Remove-NetFirewallRule -DisplayName "允许 TCP 80"

# 启用/禁用规则
Enable-NetFirewallRule -DisplayName "远程桌面*"
Disable-NetFirewallRule -DisplayName "远程桌面*"
```

### 网络连接查看

```powershell
# 查看 TCP 连接
Get-NetTCPConnection

# 查看监听端口
Get-NetTCPConnection -State Listen

# 查看已建立的连接
Get-NetTCPConnection -State Established

# 查看特定端口
Get-NetTCPConnection -LocalPort 80

# 查看连接及对应进程
Get-NetTCPConnection | Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State,@{Name="Process";Expression={(Get-Process -Id $_.OwningProcess).ProcessName}}

# 查看 UDP 端点
Get-NetUDPEndpoint
```

---

## 网络抓包工具

### 使用 netsh trace

Windows 内置的网络抓包工具。

```cmd
# 开始抓包
netsh trace start capture=yes tracefile=C:\capture.etl

# 开始抓包（指定网络适配器）
netsh trace start capture=yes tracefile=C:\capture.etl captureinterface="以太网"

# 抓取特定 IP 的流量
netsh trace start capture=yes tracefile=C:\capture.etl IPv4.Address=192.168.1.100

# 停止抓包
netsh trace stop

# 查看支持的场景
netsh trace show scenarios

# 使用特定场景抓包
netsh trace start scenario=InternetClient capture=yes tracefile=C:\capture.etl

# 转换 ETL 文件（使用 Microsoft Message Analyzer 或 Wireshark）
# 可以使用 etl2pcapng 工具转换为 pcapng 格式
```

### 使用 pktmon（Windows 10 1809+）

```cmd
# 列出所有网络组件
pktmon comp list

# 开始抓包
pktmon start --etw

# 抓包到文件
pktmon start --etw -f pktmon.etl

# 添加过滤器（特定 IP）
pktmon filter add -i 192.168.1.100

# 添加过滤器（特定端口）
pktmon filter add -p 80

# 列出过滤器
pktmon filter list

# 停止抓包
pktmon stop

# 查看统计信息
pktmon counters

# 重置计数器
pktmon reset

# 转换为 pcapng 格式（可用 Wireshark 打开）
pktmon etl2txt pktmon.etl
pktmon pcapng pktmon.etl -o pktmon.pcapng

# 实时显示抓包
pktmon start -c
```

### Wireshark

Windows 下最强大的抓包工具（需要单独安装）。

**安装方法：**
1. 下载 Wireshark: https://www.wireshark.org/download.html
2. 安装时确保安装 Npcap 或 WinPcap

**常用过滤器：**

```
# IP 过滤
ip.addr == 192.168.1.100
ip.src == 192.168.1.100
ip.dst == 192.168.1.100

# 端口过滤
tcp.port == 80
udp.port == 53

# 协议过滤
http
dns
icmp
arp

# 组合过滤
ip.addr == 192.168.1.100 && tcp.port == 80
tcp.flags.syn == 1 && tcp.flags.ack == 0

# HTTP 过滤
http.request.method == "GET"
http.host contains "baidu"
```

---

## 网络故障排查流程

### 🔍 系统化排查步骤

```
┌─────────────────────────────────────────────────────────┐
│              Windows 网络故障排查流程                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 检查网络适配器状态                                    │
│     └── 设备管理器、网络连接、指示灯                       │
│              ↓                                          │
│  2. 检查 IP 配置                                         │
│     └── ipconfig /all                                   │
│              ↓                                          │
│  3. 测试本地回环                                         │
│     └── ping 127.0.0.1                                  │
│              ↓                                          │
│  4. 测试网关连通性                                       │
│     └── ping 网关地址                                    │
│              ↓                                          │
│  5. 测试外网连通性                                       │
│     └── ping 8.8.8.8                                    │
│              ↓                                          │
│  6. 测试 DNS 解析                                        │
│     └── nslookup www.baidu.com                          │
│              ↓                                          │
│  7. 检查路由                                            │
│     └── tracert / pathping                              │
│              ↓                                          │
│  8. 检查防火墙                                          │
│     └── netsh advfirewall show currentprofile           │
│              ↓                                          │
│  9. 检查端口和服务                                       │
│     └── netstat -ano                                    │
│              ↓                                          │
│  10. 网络重置（最后手段）                                │
│      └── netsh int ip reset + netsh winsock reset      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 📋 常见问题快速诊断

| 问题现象 | 快速检查命令 | 可能原因 | 解决方案 |
|---------|-------------|---------|---------|
| 无法上网 | `ipconfig /all`<br>`ping 网关` | IP 配置错误<br>网线未连接 | 重新配置 IP<br>检查物理连接 |
| DNS 解析失败 | `nslookup www.baidu.com`<br>`ipconfig /flushdns` | DNS 服务器问题<br>DNS 缓存损坏 | 更换 DNS<br>清除缓存 |
| 网络时通时断 | `pathping 8.8.8.8`<br>`netsh int ip show config` | IP 冲突<br>网卡驱动问题 | 更换 IP<br>更新驱动 |
| 无法访问特定网站 | `tracert 目标地址`<br>`telnet IP 端口` | 路由问题<br>防火墙阻止 | 检查路由<br>调整防火墙 |
| 端口被占用 | `netstat -ano | findstr :端口`<br>`tasklist | findstr PID` | 程序冲突 | 结束进程 |
| 远程桌面连接失败 | `Test-NetConnection IP -Port 3389` | RDP 服务未启动<br>防火墙阻止 | 启动服务<br>开放端口 |

### 🛠️ 一键诊断脚本

#### 批处理脚本 (.bat)

```batch
@echo off
echo ========================================
echo Windows 网络诊断脚本
echo ========================================
echo.

echo [1] 网络适配器状态
echo ----------------------------------------
ipconfig | findstr /C:"适配器" /C:"IPv4" /C:"默认网关"
echo.

echo [2] DNS 配置
echo ----------------------------------------
ipconfig /all | findstr /C:"DNS 服务器"
echo.

echo [3] 测试本地回环
echo ----------------------------------------
ping -n 2 127.0.0.1
echo.

echo [4] 测试网关
echo ----------------------------------------
for /f "tokens=3" %%i in ('route print ^| findstr "\<0.0.0.0\>"') do set gateway=%%i
ping -n 2 %gateway%
echo.

echo [5] 测试外网
echo ----------------------------------------
ping -n 2 8.8.8.8
echo.

echo [6] DNS 解析测试
echo ----------------------------------------
nslookup www.baidu.com
echo.

echo [7] 路由表
echo ----------------------------------------
route print -4 | findstr "0.0.0.0"
echo.

echo [8] 活动连接
echo ----------------------------------------
netstat -an | findstr "ESTABLISHED"
echo.

echo [9] 监听端口
echo ----------------------------------------
netstat -an | findstr "LISTENING"
echo.

echo ========================================
echo 诊断完成
echo ========================================
pause
```

#### PowerShell 脚本 (.ps1)

```powershell
# network_diagnostic.ps1
Write-Host "========================================"
Write-Host "Windows 网络诊断脚本"
Write-Host "========================================"
Write-Host ""

Write-Host "[1] 网络适配器状态" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Get-NetAdapter | Format-Table Name, Status, LinkSpeed, MacAddress
Write-Host ""

Write-Host "[2] IP 配置" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Get-NetIPConfiguration | Format-Table InterfaceAlias, IPv4Address, IPv4DefaultGateway
Write-Host ""

Write-Host "[3] DNS 服务器" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Get-DnsClientServerAddress -AddressFamily IPv4 | Format-Table InterfaceAlias, ServerAddresses
Write-Host ""

Write-Host "[4] 测试连通性" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Write-Host "本地回环:"
Test-Connection -ComputerName 127.0.0.1 -Count 2 -Quiet
Write-Host "网关:"
$gateway = (Get-NetRoute -DestinationPrefix "0.0.0.0/0").NextHop
Test-Connection -ComputerName $gateway -Count 2 -Quiet
Write-Host "外网:"
Test-Connection -ComputerName 8.8.8.8 -Count 2 -Quiet
Write-Host ""

Write-Host "[5] DNS 解析测试" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Resolve-DnsName www.baidu.com -Type A | Select-Object Name, IPAddress
Write-Host ""

Write-Host "[6] 活动 TCP 连接" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Get-NetTCPConnection -State Established | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State -First 10
Write-Host ""

Write-Host "[7] 监听端口" -ForegroundColor Cyan
Write-Host "----------------------------------------"
Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, State -First 10
Write-Host ""

Write-Host "========================================" -ForegroundColor Green
Write-Host "诊断完成"
Write-Host "========================================" -ForegroundColor Green
```

### 网络重置完整步骤

当常规方法无法解决问题时，可以尝试完全重置网络配置：

```cmd
:: 以管理员身份运行命令提示符

:: 1. 重置 TCP/IP 栈
netsh int ip reset resetlog.txt

:: 2. 重置 Winsock 目录
netsh winsock reset

:: 3. 刷新 DNS 缓存
ipconfig /flushdns

:: 4. 释放并重新获取 IP
ipconfig /release
ipconfig /renew

:: 5. 重置防火墙（可选）
netsh advfirewall reset

:: 6. 重置代理设置
netsh winhttp reset proxy

:: 7. 重启计算机
shutdown /r /t 0
```

PowerShell 版本：

```powershell
# 以管理员身份运行 PowerShell

# 重置网络配置
netsh int ip reset
netsh winsock reset
Clear-DnsClientCache
ipconfig /release
ipconfig /renew

# 重置网络适配器
Get-NetAdapter | Restart-NetAdapter

# 重启计算机
Restart-Computer
```

---

## 高级技巧与工具

### 网络性能测试

```cmd
# 使用 PowerShell 测试下载速度
powershell -Command "& {$ProgressPreference = 'SilentlyContinue'; Measure-Command {Invoke-WebRequest -Uri 'http://speedtest.com/test.jpg' -OutFile 'test.jpg'}}"

# 查看网络适配器速度
wmic nic where netEnabled=true get name, speed
```

### 远程网络诊断

```cmd
# 查看远程主机的 MAC 地址（需要在同一网段）
arp -a 192.168.1.100

# 远程主机信息（需要相应权限）
systeminfo /s 远程主机名
```

### 网络监控

```powershell
# 实时监控网络流量
Get-NetAdapterStatistics | Select-Object Name, ReceivedBytes, SentBytes

# 循环监控
while ($true) {
    Clear-Host
    Get-NetAdapterStatistics | Format-Table Name, ReceivedBytes, SentBytes
    Start-Sleep -Seconds 1
}
```

---

<p align="center">
  <a href="README_zh.md">返回主页</a>
</p>
