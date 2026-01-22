# Network Debugging Guide

[![中文文档](https://img.shields.io/badge/文档-中文版-red.svg)](network_linux_zh.md)

This document introduces common network debugging commands and troubleshooting methods used in embedded system development.

## 📋 Table of Contents

- [Network Connectivity Testing](#network-connectivity-testing)
- [IP Conflict Detection](#ip-conflict-detection)
- [Route Tracing](#route-tracing)
- [DNS Diagnostics](#dns-diagnostics)
- [Network Interface Configuration](#network-interface-configuration)
- [Port Detection](#port-detection)
- [Packet Capture Analysis](#packet-capture-analysis)
- [Network Troubleshooting Workflow](#network-troubleshooting-workflow)

---

## Network Connectivity Testing

### ping

The `ping` command tests network connectivity by sending ICMP echo requests to check if a target host is reachable.

#### Basic Usage

```bash
# Basic ping test
ping 192.168.1.1

# Specify count
ping -c 4 192.168.1.1

# Continuous ping
ping 192.168.1.1

# Specify timeout (seconds)
ping -W 2 192.168.1.1

# Specify packet size (bytes)
ping -s 1000 192.168.1.1

# Set TTL value
ping -t 64 192.168.1.1

# Fast ping (0.2 second interval)
ping -i 0.2 192.168.1.1
```

#### Output Interpretation

```
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.523 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=0.412 ms

--- 192.168.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.412/0.467/0.523/0.055 ms
```

| Field | Meaning |
|-------|---------|
| `icmp_seq` | ICMP sequence number |
| `ttl` | Time to live, decrements by 1 for each router |
| `time` | Round-trip time (RTT) |
| `packet loss` | Packet loss rate |
| `rtt min/avg/max` | Minimum/average/maximum latency |

#### Common Issue Diagnosis

| Symptom | Possible Cause |
|---------|----------------|
| `Destination Host Unreachable` | Target unreachable, check gateway or routing |
| `Request timeout` | Timeout, target may be down or blocked by firewall |
| High packet loss | Network congestion or unstable link |
| High latency variance | Poor network quality or interference |

---

## IP Conflict Detection

### arp

The `arp` command views and manages the ARP cache table, useful for detecting IP address conflicts.

#### Basic Usage

```bash
# View ARP cache table
arp -a

# View ARP table for specific interface (Linux)
arp -i eth0

# Delete ARP cache entry
arp -d 192.168.1.100

# Manually add static ARP entry
arp -s 192.168.1.100 00:11:22:33:44:55
```

#### Using arping to Detect IP Conflicts

```bash
# Detect IP conflict (Linux)
arping -D -I eth0 192.168.1.100

# Send ARP request
arping -c 3 192.168.1.1
```

#### Using ip neigh (Modern Linux)

```bash
# View ARP table
ip neigh show

# Delete entry
ip neigh del 192.168.1.100 dev eth0

# Add static entry
ip neigh add 192.168.1.100 lladdr 00:11:22:33:44:55 dev eth0
```

#### IP Conflict Detection Methods

1. **Observe ARP table changes**
   ```bash
   # Clear ARP cache and observe
   ip neigh flush all
   arp -a
   ```

2. **Check if same IP maps to multiple MACs**
   ```bash
   arp -a | grep "192.168.1.100"
   ```

3. **Use arping for active probing**
   ```bash
   arping -D -c 3 -I eth0 192.168.1.100
   # Returns 0: No conflict
   # Returns 1: Conflict exists
   ```

---

## Route Tracing

### traceroute

Traces the route nodes that packets pass through to reach the target host.

#### Basic Usage

```bash
# Basic route trace
traceroute 8.8.8.8

# Use ICMP (requires root)
traceroute -I 8.8.8.8

# Use TCP
traceroute -T -p 80 8.8.8.8

# Specify maximum hops
traceroute -m 20 8.8.8.8

# Don't resolve hostnames (faster)
traceroute -n 8.8.8.8
```

#### Output Interpretation

```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  0.523 ms  0.412 ms  0.398 ms
 2  10.0.0.1 (10.0.0.1)  5.234 ms  5.123 ms  5.089 ms
 3  * * *
 4  8.8.8.8 (8.8.8.8)  20.123 ms  20.089 ms  20.045 ms
```

| Symbol | Meaning |
|--------|---------|
| `*` | No response from this hop (may be blocked by firewall) |
| Three time values | Round-trip time for three probes |

### mtr (Recommended)

`mtr` combines `ping` and `traceroute` functionality, providing real-time network quality analysis.

```bash
# Basic usage
mtr 8.8.8.8

# Report mode
mtr -r -c 100 8.8.8.8

# Don't resolve hostnames
mtr -n 8.8.8.8
```

---

## DNS Diagnostics

### nslookup

```bash
# Basic domain resolution
nslookup www.example.com

# Specify DNS server
nslookup www.example.com 8.8.8.8

# Query specific record types
nslookup -type=MX example.com
nslookup -type=A example.com
nslookup -type=AAAA example.com
nslookup -type=CNAME example.com
```

### dig (Recommended)

```bash
# Basic query
dig www.example.com

# Short output
dig +short www.example.com

# Specify DNS server
dig @8.8.8.8 www.example.com

# Query specific records
dig www.example.com A
dig www.example.com MX
dig www.example.com TXT

# Trace resolution process
dig +trace www.example.com

# Reverse lookup
dig -x 8.8.8.8
```

### host

```bash
# Simple query
host www.example.com

# Specify DNS server
host www.example.com 8.8.8.8

# Query all records
host -a example.com
```

---

## Network Interface Configuration

### ifconfig (Traditional)

```bash
# View all interfaces
ifconfig -a

# View specific interface
ifconfig eth0

# Configure IP address
ifconfig eth0 192.168.1.100 netmask 255.255.255.0

# Enable/disable interface
ifconfig eth0 up
ifconfig eth0 down

# Set MAC address
ifconfig eth0 hw ether 00:11:22:33:44:55
```

### ip (Modern Linux, Recommended)

```bash
# View all interfaces
ip addr show
ip a

# View specific interface
ip addr show eth0

# Configure IP address
ip addr add 192.168.1.100/24 dev eth0

# Delete IP address
ip addr del 192.168.1.100/24 dev eth0

# Enable/disable interface
ip link set eth0 up
ip link set eth0 down

# View routing table
ip route show
ip r

# Add default gateway
ip route add default via 192.168.1.1

# Add static route
ip route add 10.0.0.0/8 via 192.168.1.1

# View link status
ip link show
```

### ethtool (Network Card Diagnostics)

```bash
# View NIC information
ethtool eth0

# Check link status
ethtool eth0 | grep "Link detected"

# View NIC statistics
ethtool -S eth0

# Set NIC speed
ethtool -s eth0 speed 100 duplex full
```

---

## Port Detection

### netstat

```bash
# View all connections
netstat -a

# View listening ports
netstat -l

# Show TCP connections
netstat -t

# Show UDP connections
netstat -u

# Show process information
netstat -p

# Don't resolve hostnames
netstat -n

# Common combination: view all TCP listening ports
netstat -tlnp

# Check specific port
netstat -an | grep :80
```

### ss (Modern Linux, Recommended)

```bash
# View all connections
ss -a

# View listening ports
ss -l

# View TCP connections
ss -t

# View UDP connections
ss -u

# Show process information
ss -p

# Common combination
ss -tlnp

# Check specific port
ss -an | grep :80

# View connection statistics
ss -s
```

### telnet / nc (Port Connectivity Testing)

```bash
# Test port with telnet
telnet 192.168.1.1 80

# Test port with nc (netcat)
nc -zv 192.168.1.1 80

# Scan port range
nc -zv 192.168.1.1 20-100

# Test UDP port
nc -zuv 192.168.1.1 53
```

### lsof

```bash
# Check port usage
lsof -i :80

# View network connections for specific process
lsof -i -p 1234

# View all network connections
lsof -i
```

---

## Packet Capture Analysis

### tcpdump

```bash
# Capture packets on specific interface
tcpdump -i eth0

# Capture packets for specific host
tcpdump host 192.168.1.100

# Capture packets on specific port
tcpdump port 80

# Capture specific protocol
tcpdump icmp
tcpdump tcp
tcpdump udp

# Combine conditions
tcpdump -i eth0 host 192.168.1.100 and port 80

# Save to file
tcpdump -i eth0 -w capture.pcap

# Read capture file
tcpdump -r capture.pcap

# Show verbose information
tcpdump -v
tcpdump -vv

# Show packet content (hexadecimal)
tcpdump -X

# Don't resolve hostnames and ports
tcpdump -nn

# Limit capture count
tcpdump -c 100

# Common combination
tcpdump -i eth0 -nn -X -c 100 host 192.168.1.100
```

### Wireshark (GUI)

Suitable for desktop environments, supports rich protocol analysis and filtering.

Common filter expressions:

```
# IP filtering
ip.addr == 192.168.1.100
ip.src == 192.168.1.100
ip.dst == 192.168.1.100

# Port filtering
tcp.port == 80
udp.port == 53

# Protocol filtering
http
dns
icmp

# Combined filtering
ip.addr == 192.168.1.100 && tcp.port == 80
```

---

## Network Troubleshooting Workflow

### 🔍 Troubleshooting Steps

```
┌─────────────────────────────────────────────────────────┐
│              Network Troubleshooting Workflow           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Check Physical Connection                           │
│     └── Cable, indicator lights, interface status       │
│              ↓                                          │
│  2. Check Local Network Configuration                   │
│     └── IP, subnet mask, gateway, DNS                   │
│              ↓                                          │
│  3. Test Local Loopback                                 │
│     └── ping 127.0.0.1                                  │
│              ↓                                          │
│  4. Test Gateway Connectivity                           │
│     └── ping gateway address                            │
│              ↓                                          │
│  5. Test External Network                               │
│     └── ping 8.8.8.8                                    │
│              ↓                                          │
│  6. Test DNS Resolution                                 │
│     └── ping www.google.com                             │
│              ↓                                          │
│  7. Check Routing                                       │
│     └── traceroute target address                       │
│              ↓                                          │
│  8. Packet Capture Analysis                             │
│     └── tcpdump / Wireshark                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 📋 Common Issue Troubleshooting Table

| Symptom | Direction | Commands |
|---------|-----------|----------|
| Cannot ping gateway | Check IP config, cable connection | `ip addr`, `ethtool eth0` |
| Can ping IP but not domain | DNS configuration issue | `cat /etc/resolv.conf`, `dig` |
| Intermittent connectivity | Unstable link, IP conflict | `mtr`, `arping` |
| Connection timeout | Firewall, port not open | `iptables -L`, `ss -tlnp` |
| Slow network | Bandwidth limit, congestion | `iperf3`, `ethtool -S` |
| Cannot access specific port | Service not running, firewall | `ss -tlnp`, `telnet` |

### 🛠️ Quick Diagnosis Script

```bash
#!/bin/bash
# network_check.sh - Network quick diagnosis script

echo "=== Network Interface Status ==="
ip addr show

echo -e "\n=== Routing Table ==="
ip route show

echo -e "\n=== DNS Configuration ==="
cat /etc/resolv.conf

echo -e "\n=== Test Local Loopback ==="
ping -c 2 127.0.0.1

echo -e "\n=== Test Gateway ==="
GATEWAY=$(ip route | grep default | awk '{print $3}')
ping -c 2 $GATEWAY

echo -e "\n=== Test External Network ==="
ping -c 2 8.8.8.8

echo -e "\n=== Test DNS Resolution ==="
ping -c 2 www.google.com

echo -e "\n=== Listening Ports ==="
ss -tlnp
```

---

## Embedded Network Debugging Considerations

### Resource-Constrained Environments

In embedded systems, some commands may be unavailable or have limited functionality:

| Standard Command | Embedded Alternative |
|------------------|---------------------|
| `ifconfig` | `ip` or BusyBox `ifconfig` |
| `netstat` | `ss` or `/proc/net/*` |
| `traceroute` | BusyBox `traceroute` |
| `tcpdump` | Lightweight `tcpdump` or `dumpcap` |

### Viewing /proc Network Information

```bash
# Network interface statistics
cat /proc/net/dev

# TCP connection status
cat /proc/net/tcp

# UDP connection status
cat /proc/net/udp

# ARP table
cat /proc/net/arp

# Routing table
cat /proc/net/route
```

### Common Network Configuration Files

```bash
# Network interface configuration (Debian/Ubuntu)
/etc/network/interfaces

# DNS configuration
/etc/resolv.conf

# Hostname resolution
/etc/hosts

# Network service ports
/etc/services
```

---

<p align="center">
  <a href="README.md">Back to Home</a>
</p>
