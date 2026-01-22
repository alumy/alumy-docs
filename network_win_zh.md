# Windows 局域网调试指南（工控设备）

[![English](https://img.shields.io/badge/Docs-English-blue.svg)](network_win.md)

本文档专门针对工控设备的局域网调试，介绍 Windows 系统中常用的网络诊断命令和故障排查方法。

## 📋 目录

- [局域网连通性测试](#局域网连通性测试)
- [IP 配置与管理](#ip-配置与管理)
- [设备发现与 ARP](#设备发现与-arp)
- [端口连通性测试](#端口连通性测试)
- [局域网故障排查](#局域网故障排查)
- [静态 IP 配置 (netsh)](#静态-ip-配置-netsh)
- [防火墙快速配置](#防火墙快速配置)
- [网络抓包分析](#网络抓包分析)

---

## 局域网连通性测试

### ping - 测试设备连通性

测试 PC 与工控设备之间的网络连通性，是故障排查的第一步。

#### 基本用法

```cmd
# 测试工控设备连通性（默认发送 4 个包）
ping 192.168.1.100

# 持续监控设备在线状态（按 Ctrl+C 停止）
ping -t 192.168.1.100

# 指定发送次数
ping -n 10 192.168.1.100

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

#### 工控设备调试场景

```cmd
# 场景1：检查设备是否在线
ping -n 1 192.168.1.100
# 返回成功 → 设备在线
# 返回超时 → 检查网线、IP配置、设备电源

# 场景2：测试网络稳定性（检查丢包）
ping -t -l 1000 192.168.1.100
# 观察是否有丢包或延迟波动

# 场景3：批量检查多个设备
for /L %i in (1,1,254) do @ping -n 1 -w 100 192.168.1.%i | find "回复" && echo 192.168.1.%i 在线

# 场景4：检查是否存在 IP 冲突
# 先 ping 目标 IP，然后检查 ARP 表
ping 192.168.1.100
arp -a | findstr "192.168.1.100"
# 如果 MAC 地址与预期不符，可能存在 IP 冲突
```

---

## IP 配置与管理

### ipconfig - 查看和管理 IP 配置

工控设备通常使用静态 IP 地址，需要确保 PC 与设备在同一网段。

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

#### 工控设备调试场景

```cmd
# 场景1：检查 PC 的 IP 配置
ipconfig
# 确认 IPv4 地址、子网掩码与设备在同一网段
# 示例：PC: 192.168.1.10/255.255.255.0
#       设备: 192.168.1.100/255.255.255.0

# 场景2：快速查看当前网卡 IP
ipconfig | findstr /C:"IPv4" /C:"子网掩码"

# 场景3：查看网卡 MAC 地址（用于设备白名单）
ipconfig /all | findstr /C:"物理地址"

# 场景4：检查是否有多个网卡（避免网关冲突）
ipconfig | findstr /C:"适配器" /C:"IPv4"

# 场景5：网段不匹配时的判断
# PC: 192.168.1.10，设备: 192.168.0.100
# 两者不在同一网段，需要修改其中一个的 IP 配置
```

#### IP 网段计算

```cmd
# 常用网段配置：
# 配置1: IP: 192.168.1.X, 掩码: 255.255.255.0 (可用: 192.168.1.1-254)
# 配置2: IP: 192.168.0.X, 掩码: 255.255.255.0 (可用: 192.168.0.1-254)
# 配置3: IP: 10.0.0.X, 掩码: 255.255.255.0 (可用: 10.0.0.1-254)

# 判断是否同网段：
# IP 地址与子网掩码进行 AND 运算，结果相同则在同一网段
# 示例：
#   PC:   192.168.1.10  & 255.255.255.0 = 192.168.1.0
#   设备: 192.168.1.100 & 255.255.255.0 = 192.168.1.0  ✓ 同网段
#   设备: 192.168.2.100 & 255.255.255.0 = 192.168.2.0  ✗ 不同网段
```

---

## 设备发现与 ARP

### arp - 查看局域网设备

ARP（地址解析协议）缓存记录了 IP 地址与 MAC 地址的对应关系，用于发现局域网内的设备。

#### 基本用法

```cmd
# 查看 ARP 缓存表（局域网内所有通信过的设备）
arp -a

# 查看特定 IP 的 ARP 条目
arp -a 192.168.1.100

# 查看特定网段的设备
arp -a | findstr "192.168.1"

# 删除 ARP 缓存（排查 IP 冲突）
arp -d 192.168.1.100

# 删除所有 ARP 缓存
arp -d *

# 添加静态 ARP 条目（防止 ARP 欺骗）
arp -s 192.168.1.100 AA-BB-CC-DD-EE-FF
```

#### ARP 表输出解读

```
接口: 192.168.1.10 --- 0x2
  Internet 地址     物理地址              类型
  192.168.1.1       11-22-33-44-55-66     动态
  192.168.1.100     AA-BB-CC-DD-EE-FF     动态
  192.168.1.255     ff-ff-ff-ff-ff-ff     静态
```

| 字段 | 含义 |
|------|------|
| `Internet 地址` | 设备的 IP 地址 |
| `物理地址` | 设备的 MAC 地址 |
| `类型` | 动态（自动学习）或静态（手动添加）|

#### 工控设备发现场景

```cmd
# 场景1：发现局域网内的所有设备
# 先批量 ping，激活 ARP 缓存
for /L %i in (1,1,254) do @ping -n 1 -w 100 192.168.1.%i >nul
# 然后查看 ARP 表
arp -a

# 场景2：确认设备 MAC 地址（防止 IP 冲突）
ping 192.168.1.100
arp -a 192.168.1.100
# 对比设备标签上的 MAC 地址是否一致

# 场景3：检测 IP 冲突
# 如果 ping 通，但 ARP 显示的 MAC 地址与预期不符
# 说明该 IP 已被其他设备占用
arp -a | findstr "192.168.1.100"

# 场景4：清除错误的 ARP 缓存
# 当设备更换 IP 后，旧的 ARP 缓存可能导致通信失败
arp -d *
ping 192.168.1.100

# 场景5：导出 ARP 表（记录设备清单）
arp -a > device_list.txt
```

### getmac - 查看本机 MAC 地址

```cmd
# 查看本机所有网卡 MAC 地址
getmac

# 详细格式（包含网卡名称）
getmac /v

# 表格格式输出
getmac /fo table
```

### 设备发现工具命令

```cmd
# 方法1：批量 ping + ARP（最常用）
@echo off
echo 正在扫描 192.168.1.0/24 网段...
for /L %%i in (1,1,254) do (
    ping -n 1 -w 100 192.168.1.%%i >nul 2>&1
    if not errorlevel 1 echo 192.168.1.%%i 在线
)
echo.
echo ARP 缓存表：
arp -a

# 方法2：使用 netstat 查看当前连接的设备
netstat -an | findstr "ESTABLISHED"

# 方法3：查看网络邻居（仅局域网）
net view
```

---

## 端口连通性测试

### netstat - 查看网络连接和端口

用于检查工控设备的通信端口和连接状态。

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

# 查看特定端口（工控设备常用）
netstat -ano | findstr :502     # Modbus TCP
netstat -ano | findstr :44818   # EtherNet/IP
netstat -ano | findstr :2404    # IEC 61850
netstat -ano | findstr :102     # S7 协议

# 查看与特定设备的连接
netstat -an | findstr "192.168.1.100"

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

#### 工控设备调试场景

```cmd
# 场景1：检查设备连接状态
netstat -an | findstr "192.168.1.100"
# 查看与设备的连接是否建立（ESTABLISHED）

# 场景2：检查工控协议端口是否监听
netstat -an | findstr ":502"      # Modbus TCP
netstat -an | findstr ":44818"    # EtherNet/IP
netstat -an | findstr ":102"      # S7 协议

# 场景3：查找占用端口的程序
netstat -ano | findstr :502
# 记下最后一列的 PID，然后查看进程
tasklist | findstr "PID号"

# 场景4：结束占用端口的进程
taskkill /F /PID 1234

# 场景5：检查所有已建立的连接
netstat -an | findstr "ESTABLISHED"

# 场景6：导出当前连接状态
netstat -ano > connections.txt
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
```

### telnet - 测试设备端口

测试工控设备的端口连通性（需要启用 Telnet 客户端）。

#### 启用 Telnet

```cmd
# 方法1：使用 DISM（需要管理员权限）
dism /online /Enable-Feature /FeatureName:TelnetClient

# 方法2：通过控制面板
# 控制面板 → 程序 → 启用或关闭 Windows 功能 → 勾选 Telnet 客户端
```

#### 测试设备端口

```cmd
# 测试 Modbus TCP 端口（502）
telnet 192.168.1.100 502

# 测试 EtherNet/IP 端口（44818）
telnet 192.168.1.100 44818

# 测试 S7 协议端口（102）
telnet 192.168.1.100 102

# 测试 OPC UA 端口（4840）
telnet 192.168.1.100 4840

# 连接成功：显示空白屏幕或连接信息（端口开放）
# 连接失败：显示"无法打开到主机的连接"（端口关闭或被防火墙阻止）

# 退出 telnet：按 Ctrl+]，然后输入 quit
```

#### 工控设备端口测试场景

```cmd
# 场景1：测试设备是否开启 Modbus 服务
telnet 192.168.1.100 502
# 成功 → Modbus 服务正常
# 失败 → 检查设备配置、防火墙

# 场景2：批量测试常用工控端口
@echo off
echo 测试设备: 192.168.1.100
echo.
for %%p in (502 102 4840 44818) do (
    echo 测试端口 %%p...
    telnet 192.168.1.100 %%p 2>nul
    if errorlevel 1 (
        echo 端口 %%p: 关闭
    ) else (
        echo 端口 %%p: 开放
    )
)
```

---

## 静态 IP 配置 (netsh)

工控设备通常使用静态 IP，需要配置 PC 网卡与设备在同一网段。

### 查看网络接口

```cmd
# 查看所有网络接口名称
netsh interface show interface

# 查看当前 IP 配置
netsh interface ip show config

# 示例输出：
# "以太网" 的配置
#    DHCP 已启用:                          否
#    IP 地址:                              192.168.1.10
#    子网前缀:                             192.168.1.0/24 (掩码 255.255.255.0)
#    默认网关:                             192.168.1.1
```

### 配置静态 IP（与工控设备通信）

```cmd
# 场景1：配置静态 IP 与设备通信
# 假设设备 IP: 192.168.1.100，需要配置 PC 为同网段
netsh interface ip set address name="以太网" static 192.168.1.10 255.255.255.0 192.168.1.1

# 参数说明：
# - name="以太网"：网卡名称（通过 ipconfig 查看）
# - static：静态 IP 模式
# - 192.168.1.10：PC 的 IP 地址
# - 255.255.255.0：子网掩码
# - 192.168.1.1：默认网关（可选，局域网内可省略）

# 场景2：无网关的局域网配置（直连设备）
netsh interface ip set address name="以太网" static 192.168.1.10 255.255.255.0

# 场景3：恢复 DHCP 自动获取 IP
netsh interface ip set address name="以太网" dhcp

# 场景4：启用/禁用网络接口
netsh interface set interface "以太网" enabled
netsh interface set interface "以太网" disabled
```

### 常见网段配置示例

```cmd
# 示例1：与 192.168.1.100 的设备通信
netsh interface ip set address name="以太网" static 192.168.1.10 255.255.255.0

# 示例2：与 192.168.0.50 的设备通信
netsh interface ip set address name="以太网" static 192.168.0.10 255.255.255.0

# 示例3：与 10.0.0.100 的设备通信
netsh interface ip set address name="以太网" static 10.0.0.10 255.255.255.0

# 示例4：多个设备在不同网段（需要多块网卡或虚拟网卡）
# 网卡1：连接 192.168.1.x 设备
netsh interface ip set address name="以太网" static 192.168.1.10 255.255.255.0
# 网卡2：连接 192.168.2.x 设备
netsh interface ip set address name="以太网 2" static 192.168.2.10 255.255.255.0
```

### DNS 配置（可选）

```cmd
# 设置 DNS 服务器（通常不需要，工控设备用 IP 通信）
netsh interface ip set dns name="以太网" static 8.8.8.8
netsh interface ip add dns name="以太网" 8.8.4.4 index=2

# 恢复自动获取 DNS
netsh interface ip set dns name="以太网" dhcp
```

### 网络故障快速重置

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

## 防火墙快速配置

Windows 防火墙可能阻止与工控设备的通信，需要配置相应规则。

### 查看和关闭防火墙（临时调试）

```cmd
# 查看防火墙状态
netsh advfirewall show allprofiles state

# 临时关闭防火墙（仅用于调试，不推荐长期使用）
netsh advfirewall set allprofiles state off

# 重新启用防火墙
netsh advfirewall set allprofiles state on

# 恢复默认设置
netsh advfirewall reset
```

### 允许工控协议端口

```cmd
# 允许 Modbus TCP（端口 502）
netsh advfirewall firewall add rule name="Modbus TCP" dir=in action=allow protocol=TCP localport=502

# 允许 EtherNet/IP（端口 44818）
netsh advfirewall firewall add rule name="EtherNet/IP" dir=in action=allow protocol=TCP localport=44818

# 允许 S7 协议（端口 102）
netsh advfirewall firewall add rule name="S7" dir=in action=allow protocol=TCP localport=102

# 允许 OPC UA（端口 4840）
netsh advfirewall firewall add rule name="OPC UA" dir=in action=allow protocol=TCP localport=4840

# 允许特定 IP 的所有通信（推荐）
netsh advfirewall firewall add rule name="工控设备" dir=in action=allow remoteip=192.168.1.100

# 允许整个网段的通信
netsh advfirewall firewall add rule name="工控网段" dir=in action=allow remoteip=192.168.1.0/24
```

### 管理防火墙规则

```cmd
# 查看所有规则
netsh advfirewall firewall show rule name=all

# 查看特定规则
netsh advfirewall firewall show rule name="Modbus TCP"

# 删除规则
netsh advfirewall firewall delete rule name="Modbus TCP"

# 启用/禁用规则
netsh advfirewall firewall set rule name="Modbus TCP" new enable=yes
netsh advfirewall firewall set rule name="Modbus TCP" new enable=no
```

### 快速配置脚本

创建一个批处理文件 `firewall_setup.bat`（以管理员身份运行）：

```batch
@echo off
echo 正在配置工控设备防火墙规则...

REM 允许常用工控协议端口
netsh advfirewall firewall add rule name="Modbus TCP" dir=in action=allow protocol=TCP localport=502
netsh advfirewall firewall add rule name="EtherNet/IP" dir=in action=allow protocol=TCP localport=44818
netsh advfirewall firewall add rule name="S7" dir=in action=allow protocol=TCP localport=102
netsh advfirewall firewall add rule name="OPC UA" dir=in action=allow protocol=TCP localport=4840
netsh advfirewall firewall add rule name="PROFINET" dir=in action=allow protocol=TCP localport=34962-34964

REM 允许工控网段
netsh advfirewall firewall add rule name="工控网段" dir=in action=allow remoteip=192.168.1.0/24

echo 防火墙规则配置完成！
pause
```

---

## 网络抓包分析

### Wireshark

Windows 下最强大的网络抓包工具（需要单独安装）。

**安装方法：**
1. 下载 Wireshark: https://www.wireshark.org/download.html
2. 安装时确保安装 Npcap 或 WinPcap

**基本使用：**

1. 选择网络接口开始抓包
2. 使用过滤器筛选感兴趣的流量
3. 停止抓包并分析数据
4. 保存为 `.pcapng` 格式方便后续分析

**常用显示过滤器：**

```
# IP 地址过滤
ip.addr == 192.168.1.100          # 包含此 IP 的所有流量
ip.src == 192.168.1.100           # 源 IP
ip.dst == 192.168.1.100           # 目标 IP
ip.addr == 192.168.1.0/24         # 整个网段

# 端口过滤
tcp.port == 80                    # TCP 端口 80
tcp.port == 80 || tcp.port == 443 # 80 或 443 端口
tcp.dstport == 3389               # 目标端口 3389
tcp.srcport == 1234               # 源端口 1234
udp.port == 53                    # UDP 端口 53

# 协议过滤
http                              # HTTP 协议
https                             # HTTPS/TLS 协议
dns                               # DNS 协议
icmp                              # ICMP 协议
arp                               # ARP 协议
tcp                               # 所有 TCP 流量
udp                               # 所有 UDP 流量

# TCP 标志过滤
tcp.flags.syn == 1                # SYN 包
tcp.flags.syn == 1 && tcp.flags.ack == 0  # SYN 包（三次握手第一步）
tcp.flags.reset == 1              # RST 包
tcp.flags.fin == 1                # FIN 包

# HTTP 详细过滤
http.request.method == "GET"      # GET 请求
http.request.method == "POST"     # POST 请求
http.host contains "baidu"        # 主机名包含 baidu
http.request.uri contains "api"   # URI 包含 api
http.response.code == 200         # HTTP 200 响应
http.response.code >= 400         # HTTP 错误（4xx/5xx）

# DNS 过滤
dns.qry.name contains "google"    # DNS 查询包含 google
dns.flags.response == 1           # DNS 响应

# 组合过滤
ip.addr == 192.168.1.100 && tcp.port == 80
ip.src == 192.168.1.100 && http
tcp.port == 443 && ip.addr == 8.8.8.8

# 排除过滤
!arp                              # 排除 ARP
!(ip.addr == 192.168.1.1)         # 排除特定 IP
tcp.port != 22                    # 排除 SSH 流量

# 包长度过滤
frame.len > 1000                  # 大于 1000 字节的包
tcp.len > 0                       # 有数据的 TCP 包（非 ACK）

# 时间过滤
frame.time >= "2024-01-01 00:00:00"
```

**实用技巧：**

```
# 追踪 TCP 流
右键数据包 → Follow → TCP Stream

# 查看统计信息
Statistics → Protocol Hierarchy    # 协议层次统计
Statistics → Conversations         # 会话统计
Statistics → Endpoints             # 端点统计
Statistics → IO Graph              # 流量图表

# 导出对象
File → Export Objects → HTTP       # 导出 HTTP 对象（如图片、文件）

# 抓包过滤（Capture Filter，BPF 语法）
host 192.168.1.100                 # 只抓特定主机
port 80                            # 只抓 80 端口
tcp port 80 or tcp port 443        # 抓 80 或 443 端口
not arp and not icmp               # 排除 ARP 和 ICMP
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
│  2. 检查 PC 的 IP 配置                                     │
│     └── ipconfig                                          │
│     └── 确认与设备在同一网段                               │
│              ↓                                            │
│  3. 测试设备连通性                                         │
│     └── ping 设备IP                                       │
│              ↓                                            │
│  4. 检查 ARP 表（确认 MAC 地址）                           │
│     └── arp -a | findstr "设备IP"                         │
│     └── 检测 IP 冲突                                       │
│              ↓                                            │
│  5. 测试设备端口                                          │
│     └── telnet 设备IP 端口                                │
│     └── netstat -an | findstr "设备IP"                    │
│              ↓                                            │
│  6. 检查防火墙                                            │
│     └── netsh advfirewall show allprofiles state          │
│     └── 临时关闭测试：netsh advfirewall set allprofiles state off │
│              ↓                                            │
│  7. 使用 Wireshark 抓包分析                               │
│     └── 查看是否有数据包交互                               │
│              ↓                                            │
│  8. 网络重置（最后手段）                                   │
│     └── netsh int ip reset                                │
│     └── netsh winsock reset                               │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 📋 常见问题快速诊断

| 问题现象 | 快速检查命令 | 可能原因 | 解决方案 |
|---------|-------------|---------|---------|
| ping 不通设备 | `ipconfig`<br>`ping 设备IP` | 不在同一网段<br>网线故障<br>设备未开机 | 修改 IP 配置<br>检查物理连接<br>检查设备电源 |
| IP 冲突 | `arp -a \| findstr "设备IP"`<br>`arp -d *` | 多个设备使用同一 IP | 修改设备或其他主机 IP<br>清除 ARP 缓存 |
| 端口连接失败 | `telnet 设备IP 端口`<br>`netstat -an` | 设备端口未开放<br>防火墙阻止 | 检查设备配置<br>关闭防火墙测试 |
| 网段不匹配 | `ipconfig`<br>`ping 设备IP` | PC 与设备不在同一网段 | 修改 PC 或设备 IP 配置 |
| MAC 地址不匹配 | `arp -a \| findstr "设备IP"`<br>`getmac` | IP 被其他设备占用 | 检查网络内所有设备<br>修改 IP 地址 |
| 通信协议错误 | Wireshark 抓包<br>`netstat -an` | 协议端口错误<br>协议配置错误 | 确认正确的协议端口<br>检查协议参数 |

### 🛠️ 工控设备诊断脚本

保存以下脚本为 `device_check.bat`（以管理员身份运行）：

```batch
@echo off
chcp 65001 >nul
echo ========================================
echo 工控设备局域网连接诊断
echo ========================================
echo.

REM 设置设备 IP（根据实际情况修改）
set DEVICE_IP=192.168.1.100
set DEVICE_PORT=502

echo 目标设备: %DEVICE_IP%
echo 目标端口: %DEVICE_PORT%
echo ========================================
echo.

echo [1] PC 网络配置
echo ----------------------------------------
ipconfig | findstr /C:"IPv4" /C:"子网掩码"
echo.

echo [2] 测试设备连通性
echo ----------------------------------------
ping -n 4 %DEVICE_IP%
echo.

echo [3] ARP 表（检查 MAC 地址）
echo ----------------------------------------
arp -a | findstr "%DEVICE_IP%"
if errorlevel 1 (
    echo 未找到 ARP 条目，设备可能不在线
) else (
    echo 设备 MAC 地址已记录
)
echo.

echo [4] 与设备的连接状态
echo ----------------------------------------
netstat -an | findstr "%DEVICE_IP%"
if errorlevel 1 (
    echo 无活动连接
) else (
    echo 存在活动连接
)
echo.

echo [5] 本地监听端口
echo ----------------------------------------
netstat -an | findstr "LISTENING" | findstr "%DEVICE_PORT%"
if errorlevel 1 (
    echo 端口 %DEVICE_PORT% 未监听
) else (
    echo 端口 %DEVICE_PORT% 正在监听
)
echo.

echo [6] 防火墙状态
echo ----------------------------------------
netsh advfirewall show allprofiles state | findstr "State"
echo.

echo [7] 设备网段扫描
echo ----------------------------------------
echo 正在扫描 192.168.1.0/24 网段（需要1-2分钟）...
for /L %%i in (1,1,254) do (
    ping -n 1 -w 100 192.168.1.%%i >nul 2>&1
    if not errorlevel 1 echo   192.168.1.%%i 在线
)
echo.

echo ========================================
echo 诊断完成
echo ========================================
echo.
echo 建议：
echo 1. 如果 ping 不通，检查 IP 配置和物理连接
echo 2. 如果 ARP 表无记录，检查设备是否开机
echo 3. 如果端口无连接，使用 telnet 测试端口
echo 4. 如果防火墙开启，临时关闭测试
echo.
pause
```

### 快速配置 PC 网络脚本

保存以下脚本为 `set_ip.bat`（以管理员身份运行）：

```batch
@echo off
chcp 65001 >nul
echo ========================================
echo 配置 PC 网络以连接工控设备
echo ========================================
echo.

REM 显示当前网卡
echo 当前网络适配器：
netsh interface show interface
echo.

REM 设置目标网卡名称（根据实际情况修改）
set ADAPTER_NAME=以太网

echo 请选择配置方案：
echo 1. 192.168.1.x 网段（设备 IP: 192.168.1.100）
echo 2. 192.168.0.x 网段（设备 IP: 192.168.0.100）
echo 3. 10.0.0.x 网段（设备 IP: 10.0.0.100）
echo 4. 自定义
echo 5. 恢复 DHCP
echo.

set /p choice=请输入选项 (1-5): 

if "%choice%"=="1" (
    netsh interface ip set address name="%ADAPTER_NAME%" static 192.168.1.10 255.255.255.0
    echo 已配置: 192.168.1.10 / 255.255.255.0
)

if "%choice%"=="2" (
    netsh interface ip set address name="%ADAPTER_NAME%" static 192.168.0.10 255.255.255.0
    echo 已配置: 192.168.0.10 / 255.255.255.0
)

if "%choice%"=="3" (
    netsh interface ip set address name="%ADAPTER_NAME%" static 10.0.0.10 255.255.255.0
    echo 已配置: 10.0.0.10 / 255.255.255.0
)

if "%choice%"=="4" (
    set /p custom_ip=请输入 PC IP 地址: 
    set /p custom_mask=请输入子网掩码 (默认 255.255.255.0): 
    if "%custom_mask%"=="" set custom_mask=255.255.255.0
    netsh interface ip set address name="%ADAPTER_NAME%" static %custom_ip% %custom_mask%
    echo 已配置: %custom_ip% / %custom_mask%
)

if "%choice%"=="5" (
    netsh interface ip set address name="%ADAPTER_NAME%" dhcp
    echo 已恢复 DHCP 自动获取
)

echo.
echo 当前 IP 配置：
ipconfig | findstr /C:"IPv4" /C:"子网掩码"
echo.
pause
```

### 常见故障快速解决方案

#### 问题1：ping 不通设备

```cmd
REM 步骤1：检查 PC 和设备是否在同一网段
ipconfig

REM 步骤2：清除 ARP 缓存后重试
arp -d *
ping 192.168.1.100

REM 步骤3：临时关闭防火墙测试
netsh advfirewall set allprofiles state off
ping 192.168.1.100
netsh advfirewall set allprofiles state on
```

#### 问题2：IP 冲突

```cmd
REM 步骤1：查看 ARP 表中的 MAC 地址
arp -a | findstr "192.168.1.100"

REM 步骤2：清除 ARP 缓存
arp -d *

REM 步骤3：修改 PC 或设备的 IP 地址
netsh interface ip set address name="以太网" static 192.168.1.10 255.255.255.0
```

#### 问题3：端口连接失败

```cmd
REM 步骤1：测试端口是否开放
telnet 192.168.1.100 502

REM 步骤2：检查防火墙是否阻止
netsh advfirewall firewall add rule name="Modbus TCP" dir=in action=allow protocol=TCP localport=502

REM 步骤3：检查是否有程序占用端口
netstat -ano | findstr :502
```

#### 问题4：网络配置错误

```cmd
REM 完全重置网络配置（以管理员身份运行）
netsh int ip reset
netsh winsock reset
ipconfig /flushdns
shutdown /r /t 10
```

---

[返回主页](README_zh.md)
