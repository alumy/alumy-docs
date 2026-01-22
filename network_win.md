# Windows LAN Debugging Guide (Industrial Devices)

[![中文文档](https://img.shields.io/badge/文档-中文版-red.svg)](network_win_zh.md)

This document focuses on LAN debugging for industrial control devices, covering commonly used network diagnostic commands and troubleshooting methods in Windows.

## 📋 Table of Contents

- [LAN Connectivity Testing](#lan-connectivity-testing)
- [IP Configuration and Management](#ip-configuration-and-management)
- [Device Discovery and ARP](#device-discovery-and-arp)
- [Port Connectivity Testing](#port-connectivity-testing)
- [LAN Troubleshooting](#lan-troubleshooting)
- [Static IP Configuration (netsh)](#static-ip-configuration-netsh)
- [Quick Firewall Configuration](#quick-firewall-configuration)
- [Network Packet Analysis](#network-packet-analysis)

---

## LAN Connectivity Testing

### ping - Test Device Connectivity

Testing network connectivity between PC and industrial devices is the first step in troubleshooting.

#### Basic Usage

```cmd
# Test connectivity to industrial device (sends 4 packets by default)
ping 192.168.1.100

# Continuous monitoring of device online status (press Ctrl+C to stop)
ping -t 192.168.1.100

# Specify number of packets to send
ping -n 10 192.168.1.100

# Specify packet size (bytes)
ping -l 1000 192.168.1.100

# Don't allow packet fragmentation
ping -f 192.168.1.100

# Set TTL value
ping -i 64 192.168.1.100

# Set timeout (milliseconds)
ping -w 2000 192.168.1.100

# Specify source address
ping -S 192.168.1.10 192.168.1.100

# Force IPv4
ping -4 www.example.com

# Resolve address to hostname
ping -a 192.168.1.100

# IPv6 ping
ping -6 fe80::1
```

#### Output Interpretation

```
Pinging 192.168.1.100 with 32 bytes of data:
Reply from 192.168.1.100: bytes=32 time=1ms TTL=64
Reply from 192.168.1.100: bytes=32 time<1ms TTL=64
Reply from 192.168.1.100: bytes=32 time<1ms TTL=64
Reply from 192.168.1.100: bytes=32 time=1ms TTL=64

Ping statistics for 192.168.1.100:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

#### Common Error Messages

| Error Message | Meaning | Troubleshooting |
|---------------|---------|-----------------|
| `Request timed out` | Target host not responding | Check device status, firewall |
| `Destination host unreachable` | Route unavailable | Check gateway config, routing table |
| `General failure` | Network adapter issue | Check NIC driver, cable connection |
| `Destination host unreachable` | Local network config issue | Check IP configuration |
| `Destination port unreachable` | Port closed | Check firewall rules |

#### Industrial Device Debugging Scenarios

```cmd
# Scenario 1: Check if device is online
ping -n 1 192.168.1.100
# Success → Device online
# Timeout → Check cable, IP config, device power

# Scenario 2: Test network stability (check for packet loss)
ping -t -l 1000 192.168.1.100
# Monitor for packet loss or latency fluctuations

# Scenario 3: Batch check multiple devices
for /L %i in (1,1,254) do @ping -n 1 -w 100 192.168.1.%i | find "Reply" && echo 192.168.1.%i is online

# Scenario 4: Check for IP conflicts
# First ping the IP, then check ARP table
ping 192.168.1.100
arp -a | findstr "192.168.1.100"
# If MAC address doesn't match expected, may be IP conflict
```

---

## IP Configuration and Management

### ipconfig - View and Manage IP Configuration

Industrial devices typically use static IP addresses, ensure PC is on the same subnet as the device.

#### Basic Usage

```cmd
# View basic network configuration
ipconfig

# View detailed configuration
ipconfig /all

# Release DHCP-assigned IP
ipconfig /release

# Renew DHCP address
ipconfig /renew

# Flush DNS cache
ipconfig /flushdns

# Display DNS cache contents
ipconfig /displaydns

# Re-register DNS records
ipconfig /registerdns

# Release IPv6 address
ipconfig /release6

# Renew IPv6 address
ipconfig /renew6

# Display all adapter information
ipconfig /allcompartments

# View specific adapter
ipconfig | findstr "Ethernet"
```

#### Output Interpretation

```
Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : localdomain
   Link-local IPv6 Address . . . . . : fe80::a1b2:c3d4:e5f6:7890%12
   IPv4 Address. . . . . . . . . . . : 192.168.1.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

#### Industrial Device Debugging Scenarios

```cmd
# Scenario 1: Check PC IP configuration
ipconfig
# Confirm IPv4 address and subnet mask are on same subnet as device
# Example: PC: 192.168.1.10/255.255.255.0
#          Device: 192.168.1.100/255.255.255.0

# Scenario 2: Quick view current network card IP
ipconfig | findstr /C:"IPv4" /C:"Subnet"

# Scenario 3: View network card MAC address (for device whitelisting)
ipconfig /all | findstr /C:"Physical Address"

# Scenario 4: Check for multiple network cards (avoid gateway conflicts)
ipconfig | findstr /C:"adapter" /C:"IPv4"

# Scenario 5: Determine if subnets don't match
# PC: 192.168.1.10, Device: 192.168.0.100
# Not on same subnet, need to modify one IP configuration
```

#### IP Subnet Calculation

```cmd
# Common subnet configurations:
# Config 1: IP: 192.168.1.X, Mask: 255.255.255.0 (Available: 192.168.1.1-254)
# Config 2: IP: 192.168.0.X, Mask: 255.255.255.0 (Available: 192.168.0.1-254)
# Config 3: IP: 10.0.0.X, Mask: 255.255.255.0 (Available: 10.0.0.1-254)

# To determine if on same subnet:
# Perform AND operation between IP address and subnet mask, if results match, same subnet
# Example:
#   PC:   192.168.1.10  & 255.255.255.0 = 192.168.1.0
#   Device: 192.168.1.100 & 255.255.255.0 = 192.168.1.0  ✓ Same subnet
#   Device: 192.168.2.100 & 255.255.255.0 = 192.168.2.0  ✗ Different subnet
```

---

## Device Discovery and ARP

### arp - View LAN Devices

ARP (Address Resolution Protocol) cache records the mapping between IP and MAC addresses, used for discovering devices on the LAN.

#### Basic Usage

```cmd
# Display ARP cache table (all devices communicated with on LAN)
arp -a

# View ARP entry for specific IP
arp -a 192.168.1.100

# View devices on specific subnet
arp -a | findstr "192.168.1"

# Delete ARP entry (troubleshoot IP conflicts)
arp -d 192.168.1.100

# Delete all ARP cache
arp -d *

# Add static ARP entry (prevent ARP spoofing)
arp -s 192.168.1.100 AA-BB-CC-DD-EE-FF
```

#### ARP Table Output Interpretation

```
Interface: 192.168.1.10 --- 0x2
  Internet Address      Physical Address      Type
  192.168.1.1          11-22-33-44-55-66     dynamic
  192.168.1.100        AA-BB-CC-DD-EE-FF     dynamic
  192.168.1.255        ff-ff-ff-ff-ff-ff     static
```

| Field | Meaning |
|-------|---------|
| `Internet Address` | Device IP address |
| `Physical Address` | Device MAC address |
| `Type` | Dynamic (auto-learned) or Static (manually added) |

#### Industrial Device Discovery Scenarios

```cmd
# Scenario 1: Discover all devices on LAN
# First batch ping to populate ARP cache
for /L %i in (1,1,254) do @ping -n 1 -w 100 192.168.1.%i >nul
# Then view ARP table
arp -a

# Scenario 2: Confirm device MAC address (prevent IP conflicts)
ping 192.168.1.100
arp -a 192.168.1.100
# Compare with MAC address on device label

# Scenario 3: Detect IP conflict
# If ping works but ARP shows unexpected MAC address
# Indicates IP already used by another device
arp -a | findstr "192.168.1.100"

# Scenario 4: Clear incorrect ARP cache
# When device changes IP, old ARP cache may cause communication failure
arp -d *
ping 192.168.1.100

# Scenario 5: Export ARP table (record device inventory)
arp -a > device_list.txt
```

### getmac - View Local MAC Address

```cmd
# View MAC addresses of all local network cards
getmac

# Detailed format (includes network card names)
getmac /v

# Table format output
getmac /fo table
```

### Device Discovery Tool Commands

```cmd
# Method 1: Batch ping + ARP (most common)
@echo off
echo Scanning 192.168.1.0/24 subnet...
for /L %%i in (1,1,254) do (
    ping -n 1 -w 100 192.168.1.%%i >nul 2>&1
    if not errorlevel 1 echo 192.168.1.%%i is online
)
echo.
echo ARP cache table:
arp -a

# Method 2: Use netstat to view currently connected devices
netstat -an | findstr "ESTABLISHED"

# Method 3: View network neighbors (LAN only)
net view
```

---

## Port Connectivity Testing

### netstat - View Network Connections and Ports

Used to check industrial device communication ports and connection status.

#### Basic Usage

```cmd
# View all connections and listening ports
netstat -a

# View listening ports
netstat -an

# Display executables (requires administrator privileges)
netstat -ano

# Display with process ID and process name
netstat -anob

# Show TCP connections only
netstat -ant

# Show UDP connections only
netstat -anu

# Show Ethernet statistics
netstat -e

# Show routing table
netstat -r

# Show protocol statistics
netstat -s

# View specific port (industrial device common ports)
netstat -ano | findstr :502     # Modbus TCP
netstat -ano | findstr :44818   # EtherNet/IP
netstat -ano | findstr :2404    # IEC 61850
netstat -ano | findstr :102     # S7 Protocol

# View connections to specific device
netstat -an | findstr "192.168.1.100"

# View specific process connections
netstat -ano | findstr 1234
```

#### Output Interpretation

```
Proto  Local Address          Foreign Address        State           PID
TCP    0.0.0.0:80            0.0.0.0:0              LISTENING       4
TCP    192.168.1.10:1234     192.168.1.100:502      ESTABLISHED     5678
```

| State | Meaning |
|-------|---------|
| `LISTENING` | Listening, waiting for connection |
| `ESTABLISHED` | Connection established |
| `TIME_WAIT` | Connection closed, waiting for timeout |
| `CLOSE_WAIT` | Waiting to close |
| `SYN_SENT` | Connection request sent |
| `SYN_RECEIVED` | Connection request received |

#### Industrial Device Debugging Scenarios

```cmd
# Scenario 1: Check device connection status
netstat -an | findstr "192.168.1.100"
# Check if connection to device is established (ESTABLISHED)

# Scenario 2: Check if industrial protocol ports are listening
netstat -an | findstr ":502"      # Modbus TCP
netstat -an | findstr ":44818"    # EtherNet/IP
netstat -an | findstr ":102"      # S7 Protocol

# Scenario 3: Find program using port
netstat -ano | findstr :502
# Note the PID in last column, then view process
tasklist | findstr "PID"

# Scenario 4: Terminate process using port
taskkill /F /PID 1234

# Scenario 5: Check all established connections
netstat -an | findstr "ESTABLISHED"

# Scenario 6: Export current connection status
netstat -ano > connections.txt
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

### telnet - Test Device Ports

Test port connectivity of industrial devices (requires Telnet client enabled).

#### Enable Telnet

```cmd
# Method 1: Use DISM (requires administrator privileges)
dism /online /Enable-Feature /FeatureName:TelnetClient

# Method 2: Via Control Panel
# Control Panel → Programs → Turn Windows features on or off → Check Telnet Client
```

#### Test Device Ports

```cmd
# Test Modbus TCP port (502)
telnet 192.168.1.100 502

# Test EtherNet/IP port (44818)
telnet 192.168.1.100 44818

# Test S7 protocol port (102)
telnet 192.168.1.100 102

# Test OPC UA port (4840)
telnet 192.168.1.100 4840

# Connection successful: blank screen or connection info (port open)
# Connection failed: "Could not open connection to host" (port closed or blocked by firewall)

# Exit telnet: Press Ctrl+], then type quit
```

#### Industrial Device Port Testing Scenarios

```cmd
# Scenario 1: Test if device has Modbus service enabled
telnet 192.168.1.100 502
# Success → Modbus service normal
# Failure → Check device configuration, firewall

# Scenario 2: Batch test common industrial ports
@echo off
echo Testing device: 192.168.1.100
echo.
for %%p in (502 102 4840 44818) do (
    echo Testing port %%p...
    telnet 192.168.1.100 %%p 2>nul
    if errorlevel 1 (
        echo Port %%p: Closed
    ) else (
        echo Port %%p: Open
    )
)
```

---

## Static IP Configuration (netsh)

Industrial devices typically use static IPs, need to configure PC network card to be on same subnet as device.

### View Network Interfaces

```cmd
# View all network interface names
netsh interface show interface

# View current IP configuration
netsh interface ip show config

# Example output:
# Configuration for interface "Ethernet"
#    DHCP enabled:                         No
#    IP Address:                           192.168.1.10
#    Subnet Prefix:                        192.168.1.0/24 (mask 255.255.255.0)
#    Default Gateway:                      192.168.1.1
```

### Configure Static IP (to Communicate with Industrial Devices)

```cmd
# Scenario 1: Configure static IP to communicate with device
# Assume device IP: 192.168.1.100, need to configure PC to same subnet
netsh interface ip set address name="Ethernet" static 192.168.1.10 255.255.255.0 192.168.1.1

# Parameter explanation:
# - name="Ethernet": Network card name (view via ipconfig)
# - static: Static IP mode
# - 192.168.1.10: PC IP address
# - 255.255.255.0: Subnet mask
# - 192.168.1.1: Default gateway (optional, can be omitted in LAN)

# Scenario 2: LAN configuration without gateway (direct device connection)
netsh interface ip set address name="Ethernet" static 192.168.1.10 255.255.255.0

# Scenario 3: Restore DHCP automatic IP acquisition
netsh interface ip set address name="Ethernet" dhcp

# Scenario 4: Enable/disable network interface
netsh interface set interface "Ethernet" enabled
netsh interface set interface "Ethernet" disabled
```

### Common Subnet Configuration Examples

```cmd
# Example 1: Communicate with device at 192.168.1.100
netsh interface ip set address name="Ethernet" static 192.168.1.10 255.255.255.0

# Example 2: Communicate with device at 192.168.0.50
netsh interface ip set address name="Ethernet" static 192.168.0.10 255.255.255.0

# Example 3: Communicate with device at 10.0.0.100
netsh interface ip set address name="Ethernet" static 10.0.0.10 255.255.255.0

# Example 4: Multiple devices on different subnets (requires multiple NICs or virtual NICs)
# NIC 1: Connect to 192.168.1.x devices
netsh interface ip set address name="Ethernet" static 192.168.1.10 255.255.255.0
# NIC 2: Connect to 192.168.2.x devices
netsh interface ip set address name="Ethernet 2" static 192.168.2.10 255.255.255.0
```

### DNS Configuration (Optional)

```cmd
# Set DNS server (usually not needed, industrial devices use IP communication)
netsh interface ip set dns name="Ethernet" static 8.8.8.8
netsh interface ip add dns name="Ethernet" 8.8.4.4 index=2

# Restore automatic DNS acquisition
netsh interface ip set dns name="Ethernet" dhcp
```

### Quick Network Fault Reset

```cmd
# Reset TCP/IP stack
netsh int ip reset

# Reset Winsock catalog
netsh winsock reset

# Reset firewall configuration
netsh advfirewall reset

# Reset all network settings (requires restart)
netsh int ip reset
netsh winsock reset
ipconfig /flushdns
ipconfig /release
ipconfig /renew
```

---

## Quick Firewall Configuration

Windows Firewall may block communication with industrial devices, need to configure appropriate rules.

### View and Disable Firewall (Temporary Debugging)

```cmd
# View firewall status
netsh advfirewall show allprofiles state

# Temporarily disable firewall (for debugging only, not recommended long-term)
netsh advfirewall set allprofiles state off

# Re-enable firewall
netsh advfirewall set allprofiles state on

# Restore default settings
netsh advfirewall reset
```

### Allow Industrial Protocol Ports

```cmd
# Allow Modbus TCP (port 502)
netsh advfirewall firewall add rule name="Modbus TCP" dir=in action=allow protocol=TCP localport=502

# Allow EtherNet/IP (port 44818)
netsh advfirewall firewall add rule name="EtherNet/IP" dir=in action=allow protocol=TCP localport=44818

# Allow S7 protocol (port 102)
netsh advfirewall firewall add rule name="S7" dir=in action=allow protocol=TCP localport=102

# Allow OPC UA (port 4840)
netsh advfirewall firewall add rule name="OPC UA" dir=in action=allow protocol=TCP localport=4840

# Allow all communication from specific IP (recommended)
netsh advfirewall firewall add rule name="Industrial Device" dir=in action=allow remoteip=192.168.1.100

# Allow communication from entire subnet
netsh advfirewall firewall add rule name="Industrial Subnet" dir=in action=allow remoteip=192.168.1.0/24
```

### Manage Firewall Rules

```cmd
# View all rules
netsh advfirewall firewall show rule name=all

# View specific rule
netsh advfirewall firewall show rule name="Modbus TCP"

# Delete rule
netsh advfirewall firewall delete rule name="Modbus TCP"

# Enable/disable rule
netsh advfirewall firewall set rule name="Modbus TCP" new enable=yes
netsh advfirewall firewall set rule name="Modbus TCP" new enable=no
```

### Quick Configuration Script

Create a batch file `firewall_setup.bat` (run as administrator):

```batch
@echo off
echo Configuring firewall rules for industrial devices...

REM Allow common industrial protocol ports
netsh advfirewall firewall add rule name="Modbus TCP" dir=in action=allow protocol=TCP localport=502
netsh advfirewall firewall add rule name="EtherNet/IP" dir=in action=allow protocol=TCP localport=44818
netsh advfirewall firewall add rule name="S7" dir=in action=allow protocol=TCP localport=102
netsh advfirewall firewall add rule name="OPC UA" dir=in action=allow protocol=TCP localport=4840
netsh advfirewall firewall add rule name="PROFINET" dir=in action=allow protocol=TCP localport=34962-34964

REM Allow industrial subnet
netsh advfirewall firewall add rule name="Industrial Subnet" dir=in action=allow remoteip=192.168.1.0/24

echo Firewall rules configured!
pause
```

---

## Network Packet Analysis

### Wireshark

The most powerful network packet capture tool for Windows (requires separate installation).

**Installation:**
1. Download Wireshark: https://www.wireshark.org/download.html
2. Ensure Npcap or WinPcap is installed during setup

**Basic Usage:**

1. Select network interface to start capture
2. Use filters to narrow down traffic of interest
3. Stop capture and analyze data
4. Save as `.pcapng` format for later analysis

**Common Display Filters:**

```
# IP address filtering
ip.addr == 192.168.1.100          # All traffic with this IP
ip.src == 192.168.1.100           # Source IP
ip.dst == 192.168.1.100           # Destination IP
ip.addr == 192.168.1.0/24         # Entire subnet

# Port filtering
tcp.port == 80                    # TCP port 80
tcp.port == 80 || tcp.port == 443 # Port 80 or 443
tcp.dstport == 3389               # Destination port 3389
tcp.srcport == 1234               # Source port 1234
udp.port == 53                    # UDP port 53

# Protocol filtering
http                              # HTTP protocol
https                             # HTTPS/TLS protocol
dns                               # DNS protocol
icmp                              # ICMP protocol
arp                               # ARP protocol
tcp                               # All TCP traffic
udp                               # All UDP traffic

# TCP flag filtering
tcp.flags.syn == 1                # SYN packets
tcp.flags.syn == 1 && tcp.flags.ack == 0  # SYN packet (3-way handshake step 1)
tcp.flags.reset == 1              # RST packets
tcp.flags.fin == 1                # FIN packets

# HTTP detailed filtering
http.request.method == "GET"      # GET requests
http.request.method == "POST"     # POST requests
http.host contains "example"      # Host name contains example
http.request.uri contains "api"   # URI contains api
http.response.code == 200         # HTTP 200 response
http.response.code >= 400         # HTTP errors (4xx/5xx)

# DNS filtering
dns.qry.name contains "google"    # DNS query contains google
dns.flags.response == 1           # DNS response

# Combined filtering
ip.addr == 192.168.1.100 && tcp.port == 80
ip.src == 192.168.1.100 && http
tcp.port == 443 && ip.addr == 8.8.8.8

# Exclusion filtering
!arp                              # Exclude ARP
!(ip.addr == 192.168.1.1)         # Exclude specific IP
tcp.port != 22                    # Exclude SSH traffic

# Packet length filtering
frame.len > 1000                  # Packets larger than 1000 bytes
tcp.len > 0                       # TCP packets with data (not just ACK)

# Time filtering
frame.time >= "2024-01-01 00:00:00"
```

**Useful Tips:**

```
# Follow TCP stream
Right-click packet → Follow → TCP Stream

# View statistics
Statistics → Protocol Hierarchy    # Protocol hierarchy statistics
Statistics → Conversations         # Conversation statistics
Statistics → Endpoints             # Endpoint statistics
Statistics → IO Graph              # Traffic graph

# Export objects
File → Export Objects → HTTP       # Export HTTP objects (images, files, etc.)

# Capture filters (BPF syntax)
host 192.168.1.100                 # Only capture specific host
port 80                            # Only capture port 80
tcp port 80 or tcp port 443        # Capture port 80 or 443
not arp and not icmp               # Exclude ARP and ICMP
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
│  2. Check PC IP Configuration                             │
│     └── ipconfig                                          │
│     └── Confirm on same subnet as device                  │
│              ↓                                            │
│  3. Test Device Connectivity                              │
│     └── ping device IP                                    │
│              ↓                                            │
│  4. Check ARP Table (Confirm MAC Address)                 │
│     └── arp -a | findstr "device IP"                      │
│     └── Detect IP conflicts                               │
│              ↓                                            │
│  5. Test Device Ports                                     │
│     └── telnet device IP port                             │
│     └── netstat -an | findstr "device IP"                 │
│              ↓                                            │
│  6. Check Firewall                                        │
│     └── netsh advfirewall show allprofiles state          │
│     └── Temporarily disable: netsh advfirewall set allprofiles state off │
│              ↓                                            │
│  7. Use Wireshark for Packet Analysis                     │
│     └── Check if there's packet exchange                  │
│              ↓                                            │
│  8. Network Reset (Last Resort)                           │
│     └── netsh int ip reset                                │
│     └── netsh winsock reset                               │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 📋 Common Issues Quick Diagnosis

| Issue | Quick Check Commands | Possible Causes | Solutions |
|-------|---------------------|-----------------|-----------|
| Can't ping device | `ipconfig`<br>`ping device IP` | Not on same subnet<br>Cable fault<br>Device not powered | Modify IP config<br>Check physical connection<br>Check device power |
| IP conflict | `arp -a \| findstr "device IP"`<br>`arp -d *` | Multiple devices using same IP | Change device or other host IP<br>Clear ARP cache |
| Port connection failure | `telnet device IP port`<br>`netstat -an` | Device port not open<br>Firewall blocking | Check device config<br>Disable firewall for testing |
| Subnet mismatch | `ipconfig`<br>`ping device IP` | PC and device not on same subnet | Modify PC or device IP config |
| MAC address mismatch | `arp -a \| findstr "device IP"`<br>`getmac` | IP used by other device | Check all devices on network<br>Modify IP address |
| Protocol error | Wireshark capture<br>`netstat -an` | Wrong protocol port<br>Wrong protocol config | Confirm correct protocol port<br>Check protocol parameters |

### 🛠️ Industrial Device Diagnostic Script

Save as `device_check.bat` (run as administrator):

```batch
@echo off
chcp 65001 >nul
echo ========================================
echo Industrial Device LAN Connection Diagnostics
echo ========================================
echo.

REM Set device IP (modify as needed)
set DEVICE_IP=192.168.1.100
set DEVICE_PORT=502

echo Target device: %DEVICE_IP%
echo Target port: %DEVICE_PORT%
echo ========================================
echo.

echo [1] PC Network Configuration
echo ----------------------------------------
ipconfig | findstr /C:"IPv4" /C:"Subnet"
echo.

echo [2] Test Device Connectivity
echo ----------------------------------------
ping -n 4 %DEVICE_IP%
echo.

echo [3] ARP Table (Check MAC Address)
echo ----------------------------------------
arp -a | findstr "%DEVICE_IP%"
if errorlevel 1 (
    echo No ARP entry found, device may be offline
) else (
    echo Device MAC address recorded
)
echo.

echo [4] Connection Status with Device
echo ----------------------------------------
netstat -an | findstr "%DEVICE_IP%"
if errorlevel 1 (
    echo No active connections
) else (
    echo Active connection exists
)
echo.

echo [5] Local Listening Ports
echo ----------------------------------------
netstat -an | findstr "LISTENING" | findstr "%DEVICE_PORT%"
if errorlevel 1 (
    echo Port %DEVICE_PORT% not listening
) else (
    echo Port %DEVICE_PORT% is listening
)
echo.

echo [6] Firewall Status
echo ----------------------------------------
netsh advfirewall show allprofiles state | findstr "State"
echo.

echo [7] Device Subnet Scan
echo ----------------------------------------
echo Scanning 192.168.1.0/24 subnet (takes 1-2 minutes)...
for /L %%i in (1,1,254) do (
    ping -n 1 -w 100 192.168.1.%%i >nul 2>&1
    if not errorlevel 1 echo   192.168.1.%%i is online
)
echo.

echo ========================================
echo Diagnostics Complete
echo ========================================
echo.
echo Recommendations:
echo 1. If can't ping, check IP config and physical connection
echo 2. If no ARP entry, check if device is powered on
echo 3. If port has no connection, use telnet to test port
echo 4. If firewall is on, temporarily disable for testing
echo.
pause
```

### Quick PC Network Configuration Script

Save as `set_ip.bat` (run as administrator):

```batch
@echo off
chcp 65001 >nul
echo ========================================
echo Configure PC Network to Connect Industrial Device
echo ========================================
echo.

REM Display current network cards
echo Current network adapters:
netsh interface show interface
echo.

REM Set target network card name (modify as needed)
set ADAPTER_NAME=Ethernet

echo Select configuration option:
echo 1. 192.168.1.x subnet (Device IP: 192.168.1.100)
echo 2. 192.168.0.x subnet (Device IP: 192.168.0.100)
echo 3. 10.0.0.x subnet (Device IP: 10.0.0.100)
echo 4. Custom
echo 5. Restore DHCP
echo.

set /p choice=Enter option (1-5): 

if "%choice%"=="1" (
    netsh interface ip set address name="%ADAPTER_NAME%" static 192.168.1.10 255.255.255.0
    echo Configured: 192.168.1.10 / 255.255.255.0
)

if "%choice%"=="2" (
    netsh interface ip set address name="%ADAPTER_NAME%" static 192.168.0.10 255.255.255.0
    echo Configured: 192.168.0.10 / 255.255.255.0
)

if "%choice%"=="3" (
    netsh interface ip set address name="%ADAPTER_NAME%" static 10.0.0.10 255.255.255.0
    echo Configured: 10.0.0.10 / 255.255.255.0
)

if "%choice%"=="4" (
    set /p custom_ip=Enter PC IP address: 
    set /p custom_mask=Enter subnet mask (default 255.255.255.0): 
    if "%custom_mask%"=="" set custom_mask=255.255.255.0
    netsh interface ip set address name="%ADAPTER_NAME%" static %custom_ip% %custom_mask%
    echo Configured: %custom_ip% / %custom_mask%
)

if "%choice%"=="5" (
    netsh interface ip set address name="%ADAPTER_NAME%" dhcp
    echo Restored DHCP automatic acquisition
)

echo.
echo Current IP configuration:
ipconfig | findstr /C:"IPv4" /C:"Subnet"
echo.
pause
```

### Common Fault Quick Solutions

#### Issue 1: Can't ping device

```cmd
REM Step 1: Check if PC and device are on same subnet
ipconfig

REM Step 2: Clear ARP cache and retry
arp -d *
ping 192.168.1.100

REM Step 3: Temporarily disable firewall for testing
netsh advfirewall set allprofiles state off
ping 192.168.1.100
netsh advfirewall set allprofiles state on
```

#### Issue 2: IP conflict

```cmd
REM Step 1: View MAC address in ARP table
arp -a | findstr "192.168.1.100"

REM Step 2: Clear ARP cache
arp -d *

REM Step 3: Modify PC or device IP address
netsh interface ip set address name="Ethernet" static 192.168.1.10 255.255.255.0
```

#### Issue 3: Port connection failure

```cmd
REM Step 1: Test if port is open
telnet 192.168.1.100 502

REM Step 2: Check if firewall is blocking
netsh advfirewall firewall add rule name="Modbus TCP" dir=in action=allow protocol=TCP localport=502

REM Step 3: Check if any program is using the port
netstat -ano | findstr :502
```

#### Issue 4: Network configuration error

```cmd
REM Complete network configuration reset (run as administrator)
netsh int ip reset
netsh winsock reset
ipconfig /flushdns
shutdown /r /t 10
```

---

[Back to Home](README.md)
