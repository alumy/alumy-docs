# Linux LAN Debugging Guide (Industrial Devices)

[![中文文档](https://img.shields.io/badge/文档-中文版-red.svg)](network_linux_zh.md)

This document focuses on LAN debugging for industrial control devices, covering commonly used network diagnostic commands and troubleshooting methods in Linux/embedded systems.

## 📋 Table of Contents

- [LAN Connectivity Testing](#lan-connectivity-testing)
- [IP Configuration and Management](#ip-configuration-and-management)
- [Device Discovery and ARP](#device-discovery-and-arp)
- [Port Connectivity Testing](#port-connectivity-testing)
- [LAN Troubleshooting](#lan-troubleshooting)
- [Static IP Configuration](#static-ip-configuration)
- [Quick Firewall Configuration](#quick-firewall-configuration)
- [Network Packet Analysis](#network-packet-analysis)

---

## LAN Connectivity Testing

### ping - Test Device Connectivity

Testing network connectivity between embedded devices and industrial devices is the first step in troubleshooting.

#### Basic Usage

```bash
# Test connectivity to industrial device (sends 4 packets)
ping -c 4 192.168.1.100

# Continuous monitoring of device online status (press Ctrl+C to stop)
ping 192.168.1.100

# Fast ping (0.2 second interval)
ping -i 0.2 192.168.1.100

# Set timeout (seconds)
ping -W 2 192.168.1.100

# Specify packet size (bytes)
ping -s 1000 192.168.1.100

# Set TTL value
ping -t 64 192.168.1.100

# Specify network interface
ping -I eth0 192.168.1.100

# Quiet mode (only show statistics)
ping -c 4 -q 192.168.1.100
```

#### Output Interpretation

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

#### Industrial Device Debugging Scenarios

```bash
# Scenario 1: Check if device is online
ping -c 1 -W 1 192.168.1.100 && echo "Device online" || echo "Device offline"

# Scenario 2: Test network stability (check packet loss)
ping -c 100 -i 0.2 192.168.1.100 | grep "packet loss"

# Scenario 3: Batch check multiple devices
for i in {1..254}; do
    ping -c 1 -W 1 192.168.1.$i >/dev/null 2>&1 && echo "192.168.1.$i online"
done

# Scenario 4: Check for IP conflicts
# First ping the IP, then check ARP table
ping -c 1 192.168.1.100
arp -n | grep 192.168.1.100

# Scenario 5: Monitor device connection status and log
ping 192.168.1.100 | while read line; do
    echo "$(date): $line" >> device_monitor.log
done
```

---

## IP Configuration and Management

### ip / ifconfig - View and Manage IP Configuration

Industrial devices typically use static IP addresses, ensure embedded device is on same subnet as industrial device.

#### Using ip Command (Recommended)

```bash
# View all network interfaces
ip addr show
# Or short form
ip a

# View specific interface
ip addr show eth0

# View interface statistics
ip -s link show eth0

# View routing table
ip route show

# View ARP table
ip neigh show
```

#### Using ifconfig Command (Traditional)

```bash
# View all interfaces
ifconfig

# View specific interface
ifconfig eth0

# View brief information
ifconfig -s
```

#### Output Interpretation

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.10/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
```

| Field | Meaning |
|-------|---------|
| `UP` | Interface enabled |
| `LOWER_UP` | Physical link connected |
| `inet 192.168.1.10/24` | IPv4 address and subnet mask |
| `link/ether` | MAC address |

#### Industrial Device Debugging Scenarios

```bash
# Scenario 1: Check device IP configuration
ip addr show eth0 | grep "inet "
# Confirm IP address and subnet mask are on same subnet as device

# Scenario 2: Check network interface status
ip link show eth0
# Confirm interface is UP and LOWER_UP (cable connected)

# Scenario 3: Quick view IP and MAC
ip -br addr show
# Display all interfaces in brief format

# Scenario 4: Check for multiple network cards
ip link show | grep "^[0-9]"

# Scenario 5: Determine if subnets don't match
# Device: 192.168.1.10/24, Industrial device: 192.168.0.100
# Not on same subnet, need to modify one IP configuration
```

#### IP Subnet Calculation

```bash
# Common subnet configurations:
# Config 1: IP: 192.168.1.X, Mask: 255.255.255.0 (/24) (Available: 192.168.1.1-254)
# Config 2: IP: 192.168.0.X, Mask: 255.255.255.0 (/24) (Available: 192.168.0.1-254)
# Config 3: IP: 10.0.0.X, Mask: 255.255.255.0 (/24) (Available: 10.0.0.1-254)

# To determine if on same subnet:
# Perform AND operation between IP address and subnet mask, if results match, same subnet
# Example:
#   Device:     192.168.1.10  & 255.255.255.0 = 192.168.1.0
#   Industrial: 192.168.1.100 & 255.255.255.0 = 192.168.1.0  ✓ Same subnet
#   Industrial: 192.168.2.100 & 255.255.255.0 = 192.168.2.0  ✗ Different subnet
```

---

## Device Discovery and ARP

### arp - View LAN Devices

ARP (Address Resolution Protocol) cache records the mapping between IP and MAC addresses, used for discovering devices on the LAN.

#### Basic Usage

```bash
# Display ARP cache table
arp -n
# Or using ip command
ip neigh show

# Display ARP cache for specific interface
arp -i eth0

# Display ARP entry for specific IP
arp -n | grep 192.168.1.100

# Add static ARP entry
arp -s 192.168.1.100 AA:BB:CC:DD:EE:FF
# Or using ip command
ip neigh add 192.168.1.100 lladdr AA:BB:CC:DD:EE:FF dev eth0

# Delete ARP entry
arp -d 192.168.1.100
# Or using ip command
ip neigh del 192.168.1.100 dev eth0

# Flush all ARP cache
ip neigh flush all
```

#### ARP Table Output Interpretation

```
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.1.1              ether   11:22:33:44:55:66   C                     eth0
192.168.1.100            ether   AA:BB:CC:DD:EE:FF   C                     eth0
```

| Field | Meaning |
|-------|---------|
| `Address` | Device IP address |
| `HWaddress` | Device MAC address |
| `Flags C` | Complete entry (confirmed) |
| `Iface` | Network interface |

#### Industrial Device Discovery Scenarios

```bash
# Scenario 1: Discover all devices on LAN
# First batch ping to populate ARP cache
for i in {1..254}; do
    ping -c 1 -W 1 192.168.1.$i >/dev/null 2>&1 &
done
wait
# Then view ARP table
arp -n

# Scenario 2: Confirm device MAC address (prevent IP conflicts)
ping -c 1 192.168.1.100
arp -n | grep 192.168.1.100
# Compare with MAC address on device label

# Scenario 3: Detect IP conflict
# If ping works but ARP shows unexpected MAC address
# Indicates IP already used by another device
arp -n | grep 192.168.1.100

# Scenario 4: Clear incorrect ARP cache
# When device changes IP, old ARP cache may cause communication failure
ip neigh flush all
ping -c 1 192.168.1.100

# Scenario 5: Export ARP table (record device inventory)
arp -n > device_list.txt
```

### Device Discovery Tools

```bash
# Method 1: Use nmap scan (needs installation)
nmap -sn 192.168.1.0/24

# Method 2: Use arp-scan (needs installation)
arp-scan --interface=eth0 192.168.1.0/24

# Method 3: Simple shell script scan
#!/bin/bash
echo "Scanning 192.168.1.0/24 subnet..."
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

## Port Connectivity Testing

### netstat / ss - View Network Connections and Ports

Used to check industrial device communication ports and connection status.

#### Using ss Command (Recommended)

```bash
# View all TCP connections
ss -tan

# View all UDP connections
ss -uan

# View listening ports
ss -tln

# View connections to specific device
ss -tan | grep 192.168.1.100

# View specific port
ss -tan | grep :502      # Modbus TCP
ss -tan | grep :44818    # EtherNet/IP

# Display process information
ss -tanp

# Statistics
ss -s
```

#### Using netstat Command (Traditional)

```bash
# View all connections
netstat -an

# View TCP connections
netstat -tan

# View listening ports
netstat -tln

# Display process information
netstat -tanp

# View specific port
netstat -tan | grep :502
```

#### Output Interpretation

```
State      Recv-Q Send-Q Local Address:Port    Peer Address:Port
ESTAB      0      0      192.168.1.10:54321   192.168.1.100:502
LISTEN     0      128    0.0.0.0:502          0.0.0.0:*
```

| State | Meaning |
|-------|---------|
| `LISTEN` | Listening, waiting for connection |
| `ESTAB` (ESTABLISHED) | Connection established |
| `TIME-WAIT` | Connection closed, waiting for timeout |
| `CLOSE-WAIT` | Waiting to close |
| `SYN-SENT` | Connection request sent |
| `SYN-RECV` | Connection request received |

#### Industrial Device Debugging Scenarios

```bash
# Scenario 1: Check device connection status
ss -tan | grep 192.168.1.100
# Check if connection to device is established (ESTAB)

# Scenario 2: Check if industrial protocol ports are listening
ss -tln | grep :502      # Modbus TCP
ss -tln | grep :44818    # EtherNet/IP
ss -tln | grep :102      # S7 Protocol

# Scenario 3: Find program using port
ss -tanp | grep :502
# Or
netstat -tanp | grep :502

# Scenario 4: Check all established connections
ss -tan state established

# Scenario 5: Export current connection status
ss -tan > connections.txt
```

#### Common Industrial Protocol Port Reference

| Protocol | Port | Description |
|----------|------|-------------|
| Modbus TCP | 502 | Modbus TCP protocol |
| EtherNet/IP | 44818 | Rockwell Automation |
| PROFINET | 34962/34963/34964 | Siemens Industrial Ethernet |
| S7 | 102 | Siemens S7 protocol |
| OPC UA | 4840 | OPC Unified Architecture |
| IEC 61850 | 102 | Substation automation |
| BACnet/IP | 47808 | Building automation |
| Fins | 9600 | Omron protocol |

### nc / telnet - Test Device Ports

Test port connectivity of industrial devices.

#### Using nc (netcat)

```bash
# Test if port is open
nc -zv 192.168.1.100 502
# -z: Scan mode (don't send data)
# -v: Verbose output

# Test Modbus TCP port
nc -zv 192.168.1.100 502

# Test EtherNet/IP port
nc -zv 192.168.1.100 44818

# Test OPC UA port
nc -zv 192.168.1.100 4840

# Test port range
nc -zv 192.168.1.100 100-1000

# Set timeout (seconds)
nc -zv -w 2 192.168.1.100 502

# Connect to port (interactive mode)
nc 192.168.1.100 502
```

#### Using telnet

```bash
# Test port connectivity
telnet 192.168.1.100 502

# If connection successful, displays:
# Connected to 192.168.1.100.
# Escape character is '^]'.

# If connection fails, displays:
# Unable to connect to remote host: Connection refused

# Exit telnet: Press Ctrl+], then type quit
```

#### Industrial Device Port Testing Scenarios

```bash
# Scenario 1: Batch test common industrial ports
#!/bin/bash
device="192.168.1.100"
ports=(502 102 4840 44818)

echo "Testing device: $device"
for port in "${ports[@]}"; do
    if nc -zv -w 2 $device $port 2>&1 | grep -q "succeeded"; then
        echo "Port $port: Open"
    else
        echo "Port $port: Closed"
    fi
done

# Scenario 2: Continuously monitor port status
while true; do
    nc -zv -w 1 192.168.1.100 502 && echo "$(date): Port 502 normal"
    sleep 5
done
```

---

## Static IP Configuration

Industrial devices typically use static IPs, need to configure embedded device network interface to be on same subnet as industrial device.

### Temporary Configuration (Lost after Reboot)

#### Using ip Command

```bash
# Configure static IP (to communicate with industrial device)
# Assume industrial device IP: 192.168.1.100, configure embedded device to same subnet
sudo ip addr add 192.168.1.10/24 dev eth0

# Enable network interface
sudo ip link set eth0 up

# Add default gateway (optional, can be omitted in LAN)
sudo ip route add default via 192.168.1.1

# Delete IP address
sudo ip addr del 192.168.1.10/24 dev eth0

# Disable network interface
sudo ip link set eth0 down
```

#### Using ifconfig Command

```bash
# Configure static IP
sudo ifconfig eth0 192.168.1.10 netmask 255.255.255.0 up

# Add default gateway
sudo route add default gw 192.168.1.1

# Disable network interface
sudo ifconfig eth0 down
```

### Permanent Configuration

#### Debian/Ubuntu System (/etc/network/interfaces)

```bash
# Edit configuration file
sudo nano /etc/network/interfaces

# Add the following content:
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
    # dns-nameservers 8.8.8.8  # Optional

# Restart network service
sudo systemctl restart networking
# Or
sudo /etc/init.d/networking restart
```

#### RHEL/CentOS System (/etc/sysconfig/network-scripts/)

```bash
# Edit configuration file
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0

# Add the following content:
DEVICE=eth0
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.1.10
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
# DNS1=8.8.8.8  # Optional

# Restart network service
sudo systemctl restart network
```

#### Using systemd-networkd

```bash
# Create configuration file
sudo nano /etc/systemd/network/eth0.network

# Add the following content:
[Match]
Name=eth0

[Network]
Address=192.168.1.10/24
Gateway=192.168.1.1
# DNS=8.8.8.8  # Optional

# Enable and restart service
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
```

### Common Subnet Configuration Examples

```bash
# Example 1: Communicate with device at 192.168.1.100
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip link set eth0 up

# Example 2: Communicate with device at 192.168.0.50
sudo ip addr add 192.168.0.10/24 dev eth0
sudo ip link set eth0 up

# Example 3: Communicate with device at 10.0.0.100
sudo ip addr add 10.0.0.10/24 dev eth0
sudo ip link set eth0 up

# Example 4: Configuration script
#!/bin/bash
INTERFACE="eth0"
IP="192.168.1.10"
NETMASK="24"

sudo ip addr flush dev $INTERFACE
sudo ip addr add $IP/$NETMASK dev $INTERFACE
sudo ip link set $INTERFACE up
echo "Configured $INTERFACE: $IP/$NETMASK"
ip addr show $INTERFACE
```

### Quick Network Fault Reset

```bash
# Restart network service
sudo systemctl restart networking        # Debian/Ubuntu
sudo systemctl restart network           # RHEL/CentOS
sudo systemctl restart systemd-networkd  # systemd

# Or restart network interface
sudo ip link set eth0 down
sudo ip link set eth0 up

# Flush IP address
sudo ip addr flush dev eth0
```

---

## Quick Firewall Configuration

Embedded Linux firewall may block communication with industrial devices, need to configure appropriate rules.

### iptables Firewall

#### View and Manage Rules

```bash
# View current rules
sudo iptables -L -n -v

# View INPUT chain rules
sudo iptables -L INPUT -n -v

# View rules (with line numbers)
sudo iptables -L INPUT -n --line-numbers
```

#### Allow Industrial Device Communication

```bash
# Allow all traffic from specific IP (recommended)
sudo iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# Allow entire subnet
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# Allow Modbus TCP (port 502)
sudo iptables -A INPUT -p tcp --dport 502 -j ACCEPT

# Allow EtherNet/IP (port 44818)
sudo iptables -A INPUT -p tcp --dport 44818 -j ACCEPT

# Allow S7 protocol (port 102)
sudo iptables -A INPUT -p tcp --dport 102 -j ACCEPT

# Allow OPC UA (port 4840)
sudo iptables -A INPUT -p tcp --dport 4840 -j ACCEPT
```

#### Delete Rules

```bash
# View rule line numbers
sudo iptables -L INPUT -n --line-numbers

# Delete rule by line number
sudo iptables -D INPUT 3

# Flush all rules (for temporary debugging)
sudo iptables -F
```

#### Save Rules

```bash
# Debian/Ubuntu
sudo iptables-save > /etc/iptables/rules.v4
sudo netfilter-persistent save

# RHEL/CentOS
sudo service iptables save
sudo iptables-save > /etc/sysconfig/iptables
```

### firewalld Firewall

```bash
# View firewall status
sudo firewall-cmd --state

# Allow specific ports
sudo firewall-cmd --permanent --add-port=502/tcp    # Modbus TCP
sudo firewall-cmd --permanent --add-port=44818/tcp  # EtherNet/IP
sudo firewall-cmd --permanent --add-port=102/tcp    # S7

# Allow specific IP
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept'

# Reload rules
sudo firewall-cmd --reload

# Temporarily stop firewall (for debugging)
sudo systemctl stop firewalld
```

### Quick Configuration Script

```bash
#!/bin/bash
# Industrial device firewall configuration script

echo "Configuring firewall rules for industrial devices..."

# Allow industrial subnet
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# Allow common industrial protocol ports
sudo iptables -A INPUT -p tcp --dport 502 -j ACCEPT    # Modbus TCP
sudo iptables -A INPUT -p tcp --dport 44818 -j ACCEPT  # EtherNet/IP
sudo iptables -A INPUT -p tcp --dport 102 -j ACCEPT    # S7
sudo iptables -A INPUT -p tcp --dport 4840 -j ACCEPT   # OPC UA

# Save rules
sudo iptables-save > /etc/iptables/rules.v4

echo "Firewall rules configured!"
```

---

## Network Packet Analysis

### tcpdump - Command-line Packet Capture Tool

Most commonly used packet capture tool in embedded Linux systems.

#### Basic Usage

```bash
# Capture packets on all interfaces
sudo tcpdump

# Capture on specific interface
sudo tcpdump -i eth0

# Capture and save to file
sudo tcpdump -i eth0 -w capture.pcap

# Capture specific number of packets
sudo tcpdump -i eth0 -c 100

# Show verbose information
sudo tcpdump -i eth0 -v
sudo tcpdump -i eth0 -vv
sudo tcpdump -i eth0 -vvv

# Don't resolve hostnames (faster)
sudo tcpdump -i eth0 -n

# Don't resolve hostnames or port names
sudo tcpdump -i eth0 -nn

# Show ASCII content
sudo tcpdump -i eth0 -A

# Show hex and ASCII
sudo tcpdump -i eth0 -X
```

#### Filters

```bash
# Filter by IP address
sudo tcpdump -i eth0 host 192.168.1.100
sudo tcpdump -i eth0 src 192.168.1.100
sudo tcpdump -i eth0 dst 192.168.1.100

# Filter by subnet
sudo tcpdump -i eth0 net 192.168.1.0/24

# Filter by port
sudo tcpdump -i eth0 port 502
sudo tcpdump -i eth0 port 502 or port 44818

# Filter by protocol
sudo tcpdump -i eth0 tcp
sudo tcpdump -i eth0 udp
sudo tcpdump -i eth0 icmp
sudo tcpdump -i eth0 arp

# Combined filters
sudo tcpdump -i eth0 'host 192.168.1.100 and port 502'
sudo tcpdump -i eth0 'tcp and dst 192.168.1.100 and port 502'

# Exclusion filters
sudo tcpdump -i eth0 'not port 22'
```

#### Industrial Device Capture Scenarios

```bash
# Scenario 1: Capture all communication with device
sudo tcpdump -i eth0 -nn host 192.168.1.100 -w device_comm.pcap

# Scenario 2: Capture Modbus TCP communication
sudo tcpdump -i eth0 -nn port 502 -w modbus.pcap

# Scenario 3: Capture and view Modbus communication in real-time
sudo tcpdump -i eth0 -nn -X port 502

# Scenario 4: Capture for specific time period
sudo timeout 60 tcpdump -i eth0 -nn host 192.168.1.100 -w capture_60s.pcap

# Scenario 5: Read and analyze pcap file
sudo tcpdump -nn -r capture.pcap
sudo tcpdump -nn -r capture.pcap 'port 502'

# Scenario 6: Statistics
sudo tcpdump -i eth0 -nn host 192.168.1.100 -c 1000 | wc -l
```

### Wireshark (GUI)

If embedded device supports GUI, can use Wireshark.

```bash
# Install Wireshark
sudo apt-get install wireshark  # Debian/Ubuntu
sudo yum install wireshark      # RHEL/CentOS

# Or capture on embedded device and transfer to PC for analysis
sudo tcpdump -i eth0 -w - | ssh user@pc-ip "wireshark -k -i -"
```

**Common Display Filters** (in Wireshark):

```
# IP filtering
ip.addr == 192.168.1.100
ip.src == 192.168.1.100
ip.dst == 192.168.1.100

# Port filtering
tcp.port == 502
modbus  # Modbus protocol filter

# Combined filtering
ip.addr == 192.168.1.100 && tcp.port == 502
```

---

## LAN Troubleshooting

### 🔍 Industrial Device Connection Troubleshooting Flow

```
┌───────────────────────────────────────────────────────────┐
│     Industrial Device LAN Connection Troubleshooting      │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  1. Check Physical Connection                             │
│     └── Cable, indicator lights, device power             │
│              ↓                                            │
│  2. Check Device IP Configuration                         │
│     └── ip addr show eth0                                 │
│     └── Confirm on same subnet as industrial device       │
│              ↓                                            │
│  3. Test Device Connectivity                              │
│     └── ping device IP                                    │
│              ↓                                            │
│  4. Check ARP Table (Confirm MAC Address)                 │
│     └── arp -n | grep device IP                           │
│     └── Detect IP conflicts                               │
│              ↓                                            │
│  5. Test Device Ports                                     │
│     └── nc -zv device IP port                             │
│     └── ss -tan | grep device IP                          │
│              ↓                                            │
│  6. Check Firewall                                        │
│     └── sudo iptables -L -n                               │
│     └── Temporarily disable: sudo iptables -F             │
│              ↓                                            │
│  7. Use tcpdump for Packet Analysis                       │
│     └── Check if there's packet exchange                  │
│              ↓                                            │
│  8. Network Reset (Last Resort)                           │
│     └── Restart network service or interface              │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 📋 Common Issues Quick Diagnosis

| Issue | Quick Check Commands | Possible Causes | Solutions |
|-------|---------------------|-----------------|-----------|
| Can't ping device | `ip addr`<br>`ping device IP` | Not on same subnet<br>Cable fault<br>Device not powered | Modify IP config<br>Check physical connection<br>Check device power |
| IP conflict | `arp -n \| grep device IP`<br>`ip neigh flush all` | Multiple devices using same IP | Change device or other host IP<br>Clear ARP cache |
| Port connection failure | `nc -zv device IP port`<br>`ss -tan` | Device port not open<br>Firewall blocking | Check device config<br>Configure firewall rules |
| Subnet mismatch | `ip addr`<br>`ping device IP` | Device and industrial device not on same subnet | Modify IP configuration |
| MAC address mismatch | `arp -n \| grep device IP`<br>`ip link show` | IP used by other device | Check all devices on network<br>Modify IP address |
| Protocol error | `tcpdump -i eth0 port 502`<br>`ss -tan` | Wrong protocol port<br>Wrong protocol config | Confirm correct protocol port<br>Check protocol parameters |

### 🛠️ Industrial Device Diagnostic Script

Save as `device_check.sh`:

```bash
#!/bin/bash

# Industrial device LAN connection diagnostic script

DEVICE_IP="192.168.1.100"
DEVICE_PORT="502"
INTERFACE="eth0"

echo "========================================"
echo "Industrial Device LAN Connection Diagnostics"
echo "========================================"
echo ""
echo "Target device: $DEVICE_IP"
echo "Target port: $DEVICE_PORT"
echo "Network interface: $INTERFACE"
echo "========================================"
echo ""

echo "[1] Device Network Configuration"
echo "----------------------------------------"
ip addr show $INTERFACE | grep "inet "
echo ""

echo "[2] Test Device Connectivity"
echo "----------------------------------------"
ping -c 4 -W 2 $DEVICE_IP
echo ""

echo "[3] ARP Table (Check MAC Address)"
echo "----------------------------------------"
arp -n | grep $DEVICE_IP
if [ $? -ne 0 ]; then
    echo "No ARP entry found, device may be offline"
fi
echo ""

echo "[4] Connection Status with Device"
echo "----------------------------------------"
ss -tan | grep $DEVICE_IP
if [ $? -ne 0 ]; then
    echo "No active connections"
fi
echo ""

echo "[5] Port Connectivity Test"
echo "----------------------------------------"
nc -zv -w 2 $DEVICE_IP $DEVICE_PORT 2>&1
echo ""

echo "[6] Firewall Rules"
echo "----------------------------------------"
sudo iptables -L INPUT -n | grep $DEVICE_IP
if [ $? -ne 0 ]; then
    echo "No firewall rules found for this device"
fi
echo ""

echo "[7] Device Subnet Scan"
echo "----------------------------------------"
echo "Scanning $(echo $DEVICE_IP | cut -d'.' -f1-3).0/24 subnet..."
for i in {1..254}; do
    ip=$(echo $DEVICE_IP | cut -d'.' -f1-3).$i
    ping -c 1 -W 1 $ip >/dev/null 2>&1 && echo "  $ip online" &
done
wait
echo ""

echo "========================================"
echo "Diagnostics Complete"
echo "========================================"
echo ""
echo "Recommendations:"
echo "1. If can't ping, check IP config and physical connection"
echo "2. If no ARP entry, check if device is powered on"
echo "3. If port has no connection, check firewall rules"
echo "4. Use tcpdump for further analysis"
```

### Quick Configuration Script

Save as `set_ip.sh`:

```bash
#!/bin/bash

# Quick network interface configuration script

INTERFACE="eth0"

echo "========================================"
echo "Configure Network Interface to Connect Industrial Device"
echo "========================================"
echo ""

echo "Current network interfaces:"
ip link show | grep "^[0-9]" | cut -d: -f2
echo ""

echo "Select configuration option:"
echo "1. 192.168.1.x subnet (Device IP: 192.168.1.100)"
echo "2. 192.168.0.x subnet (Device IP: 192.168.0.100)"
echo "3. 10.0.0.x subnet (Device IP: 10.0.0.100)"
echo "4. Custom"
echo ""

read -p "Enter option (1-4): " choice

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
        read -p "Enter IP address: " IP
        read -p "Enter subnet mask (e.g., 24): " NETMASK
        ;;
    *)
        echo "Invalid option"
        exit 1
        ;;
esac

echo ""
echo "Configuring $INTERFACE: $IP/$NETMASK"

# Flush interface
sudo ip addr flush dev $INTERFACE
# Add IP address
sudo ip addr add $IP/$NETMASK dev $INTERFACE
# Enable interface
sudo ip link set $INTERFACE up

echo ""
echo "Current IP configuration:"
ip addr show $INTERFACE | grep "inet "
echo ""
echo "Configuration complete!"
```

### Common Fault Quick Solutions

#### Issue 1: Can't ping device

```bash
# Step 1: Check if device and industrial device are on same subnet
ip addr show eth0

# Step 2: Clear ARP cache and retry
sudo ip neigh flush all
ping -c 4 192.168.1.100

# Step 3: Temporarily disable firewall for testing
sudo iptables -F
ping -c 4 192.168.1.100
```

#### Issue 2: IP conflict

```bash
# Step 1: View MAC address in ARP table
arp -n | grep 192.168.1.100

# Step 2: Clear ARP cache
sudo ip neigh flush all

# Step 3: Modify IP address
sudo ip addr flush dev eth0
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip link set eth0 up
```

#### Issue 3: Port connection failure

```bash
# Step 1: Test if port is open
nc -zv 192.168.1.100 502

# Step 2: Check if firewall is blocking
sudo iptables -L INPUT -n | grep 502

# Step 3: Add firewall rule
sudo iptables -A INPUT -p tcp --dport 502 -j ACCEPT
```

#### Issue 4: Network configuration error

```bash
# Complete network configuration reset
sudo ip addr flush dev eth0
sudo ip link set eth0 down
sudo ip link set eth0 up
sudo systemctl restart networking
```

---

[Back to Home](README.md)
