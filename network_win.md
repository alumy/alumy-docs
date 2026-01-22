# Windows Network Debugging Guide

[![中文文档](https://img.shields.io/badge/文档-中文版-red.svg)](network_win_zh.md)

This document specifically covers network debugging commands and troubleshooting methods for Windows systems.

## 📋 Table of Contents

- [Network Connectivity Testing](#network-connectivity-testing)
- [Network Configuration Management](#network-configuration-management)
- [Route Diagnostics](#route-diagnostics)
- [DNS Diagnostics](#dns-diagnostics)
- [Port and Connection Management](#port-and-connection-management)
- [Advanced Network Configuration (netsh)](#advanced-network-configuration-netsh)
- [Windows Firewall](#windows-firewall)
- [PowerShell Network Commands](#powershell-network-commands)
- [Network Packet Capture Tools](#network-packet-capture-tools)
- [Network Troubleshooting Workflow](#network-troubleshooting-workflow)

---

## Network Connectivity Testing

### ping

The `ping` command in Windows tests network connectivity.

#### Basic Usage

```cmd
# Basic ping test (sends 4 packets by default)
ping 192.168.1.1

# Specify number of packets
ping -n 10 192.168.1.1

# Continuous ping (press Ctrl+C to stop)
ping -t 192.168.1.1

# Specify packet size (bytes)
ping -l 1000 192.168.1.1

# Don't fragment packets
ping -f 192.168.1.1

# Set TTL value
ping -i 64 192.168.1.1

# Set timeout (milliseconds)
ping -w 2000 192.168.1.1

# Specify source address
ping -S 192.168.1.100 192.168.1.1

# IPv6 ping
ping -6 fe80::1
```

#### Output Interpretation

```
Pinging 192.168.1.1 with 32 bytes of data:
Reply from 192.168.1.1: bytes=32 time=1ms TTL=64
Reply from 192.168.1.1: bytes=32 time<1ms TTL=64
Reply from 192.168.1.1: bytes=32 time<1ms TTL=64
Reply from 192.168.1.1: bytes=32 time=1ms TTL=64

Ping statistics for 192.168.1.1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

#### Common Error Messages

| Error Message | Meaning | Troubleshooting |
|---------------|---------|-----------------|
| `Request timed out` | Target host not responding | Check target status, firewall |
| `Destination host unreachable` | Route unavailable | Check gateway, routing table |
| `General failure` | Network adapter issue | Check NIC driver, cable |
| `Transmit failed` | Local network config issue | Check IP configuration |

---

## Network Configuration Management

### ipconfig

`ipconfig` is the primary command for viewing and managing IP configuration in Windows.

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

# Release IPv6 address
ipconfig /release6

# Renew IPv6 address
ipconfig /renew6

# Display all compartments information
ipconfig /allcompartments

# View specific adapter
ipconfig | findstr "Ethernet"
```

#### Output Interpretation

```
Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : localdomain
   Link-local IPv6 Address . . . . . : fe80::a1b2:c3d4:e5f6:7890%12
   IPv4 Address. . . . . . . . . . . : 192.168.1.100
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

#### Quick Diagnostic Tips

```cmd
# Check if IP configuration is normal
ipconfig | findstr /C:"IPv4" /C:"Subnet Mask" /C:"Default Gateway"

# Check DNS servers
ipconfig /all | findstr /C:"DNS Servers"

# View MAC address
ipconfig /all | findstr /C:"Physical Address"

# Reset network configuration (release and renew)
ipconfig /release && ipconfig /renew && ipconfig /flushdns
```

### getmac

View MAC addresses of network adapters.

```cmd
# View all MAC addresses
getmac

# Verbose format
getmac /v

# View MAC of remote computer (requires admin rights)
getmac /s RemoteHostName

# Specify output format
getmac /fo table
getmac /fo list
getmac /fo csv
```

---

## Route Diagnostics

### tracert

Traces the route path packets take to reach a target.

#### Basic Usage

```cmd
# Basic route trace
tracert www.google.com

# Don't resolve IP addresses (faster)
tracert -d 8.8.8.8

# Specify maximum hops
tracert -h 20 8.8.8.8

# Specify timeout (milliseconds)
tracert -w 2000 8.8.8.8

# IPv6 route trace
tracert -6 ipv6.google.com
```

#### Output Interpretation

```
Tracing route to www.google.com [142.250.185.68]
over a maximum of 30 hops:

  1    <1 ms    <1 ms    <1 ms  192.168.1.1
  2     2 ms     2 ms     2 ms  10.0.0.1
  3     *        *        *     Request timed out.
  4    15 ms    15 ms    15 ms  142.250.185.68

Trace complete.
```

| Symbol/Info | Meaning |
|-------------|---------|
| `*` | No response from this hop (timeout) |
| `Request timed out` | Router didn't respond to probe |
| Three time values | Round-trip time for three probes |

### pathping

Combines `ping` and `tracert`, providing detailed route quality analysis.

```cmd
# Basic usage
pathping www.google.com

# Don't resolve IP addresses
pathping -n 8.8.8.8

# Specify maximum hops
pathping -h 20 8.8.8.8

# Specify query count (default 100)
pathping -q 50 8.8.8.8

# Specify query interval (milliseconds, default 250)
pathping -p 500 8.8.8.8
```

### route

View and manage routing table.

```cmd
# View routing table
route print

# View IPv4 routing table
route print -4

# View IPv6 routing table
route print -6

# Add static route
route add 10.0.0.0 mask 255.0.0.0 192.168.1.1

# Delete route
route delete 10.0.0.0

# Add persistent route (survives reboot)
route -p add 10.0.0.0 mask 255.0.0.0 192.168.1.1

# Modify route
route change 10.0.0.0 mask 255.0.0.0 192.168.1.2

# Specify metric
route add 10.0.0.0 mask 255.0.0.0 192.168.1.1 metric 10
```

---

## DNS Diagnostics

### nslookup

DNS query tool.

#### Basic Usage

```cmd
# Basic domain resolution
nslookup www.google.com

# Specify DNS server
nslookup www.google.com 8.8.8.8

# Query specific record types
nslookup -type=A www.google.com
nslookup -type=MX google.com
nslookup -type=NS google.com
nslookup -type=CNAME www.google.com
nslookup -type=TXT google.com

# Reverse DNS lookup
nslookup 8.8.8.8

# Interactive mode
nslookup
> set type=MX
> google.com
> exit
```

#### Output Interpretation

```
Server:  dns.google
Address:  8.8.8.8

Non-authoritative answer:
Name:    www.google.com
Addresses:  142.250.185.68
          142.250.185.69
```

| Field | Meaning |
|-------|---------|
| `Server` | DNS server being used |
| `Non-authoritative answer` | From cache, not authoritative server |
| `Name` | Domain name |
| `Addresses` | Resolved IP addresses |

### Common DNS Troubleshooting

```cmd
# 1. Check DNS server configuration
ipconfig /all | findstr /C:"DNS Servers"

# 2. Clear DNS cache
ipconfig /flushdns

# 3. Test different DNS servers
nslookup www.google.com 8.8.8.8
nslookup www.google.com 1.1.1.1
nslookup www.google.com 208.67.222.222

# 4. Check hosts file
notepad C:\Windows\System32\drivers\etc\hosts

# 5. Check DNS Client service
sc query Dnscache
net start Dnscache
```

---

## Port and Connection Management

### netstat

View network connections, routing tables, interface statistics, etc.

#### Basic Usage

```cmd
# View all connections and listening ports
netstat -a

# Show numeric addresses
netstat -an

# Show with process IDs (requires admin)
netstat -ano

# Show with process IDs and names (requires admin)
netstat -anob

# Show only TCP connections
netstat -ant

# Show only UDP connections
netstat -anu

# Show Ethernet statistics
netstat -e

# Show routing table
netstat -r

# Show protocol statistics
netstat -s

# Refresh every N seconds
netstat -an 5

# Check specific port
netstat -ano | findstr :80
netstat -ano | findstr :3389

# Check specific process connections
netstat -ano | findstr 1234
```

#### Output Interpretation

```
Proto  Local Address          Foreign Address        State           PID
TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
TCP    192.168.1.100:1234     8.8.8.8:443            ESTABLISHED     5678
```

| State | Meaning |
|-------|---------|
| `LISTENING` | Waiting for connections |
| `ESTABLISHED` | Connection established |
| `TIME_WAIT` | Connection closed, waiting for timeout |
| `CLOSE_WAIT` | Waiting to close |
| `SYN_SENT` | Connection request sent |
| `SYN_RECEIVED` | Connection request received |

#### Practical Examples

```cmd
# Find process using specific port
netstat -ano | findstr :80
tasklist | findstr "1234"

# Kill process using a port
taskkill /F /PID 1234

# Count established connections
netstat -an | find /c "ESTABLISHED"

# View listening ports
netstat -an | find "LISTENING"

# View all external connections
netstat -an | find "ESTABLISHED" | find /v "127.0.0.1"
```

### telnet

Test port connectivity (requires Telnet Client to be enabled).

```cmd
# Enable Telnet Client (requires admin)
dism /online /Enable-Feature /FeatureName:TelnetClient

# Test port
telnet 192.168.1.1 80
telnet www.google.com 443

# Success shows blank screen or connection info
# Failure shows "Could not open connection to the host"
```

### Test-NetConnection (PowerShell)

```powershell
# Test connection (like ping)
Test-NetConnection 192.168.1.1

# Test specific port
Test-NetConnection 192.168.1.1 -Port 80

# Show detailed information
Test-NetConnection www.google.com -Port 443 -InformationLevel Detailed

# Route trace
Test-NetConnection www.google.com -TraceRoute
```

---

## Advanced Network Configuration (netsh)

`netsh` is Windows' powerful command-line network configuration tool.

### Network Interface Configuration

```cmd
# View all network interfaces
netsh interface show interface

# View IP configuration
netsh interface ip show config

# Set static IP
netsh interface ip set address name="Ethernet" static 192.168.1.100 255.255.255.0 192.168.1.1

# Set DHCP
netsh interface ip set address name="Ethernet" dhcp

# Set DNS servers
netsh interface ip set dns name="Ethernet" static 8.8.8.8
netsh interface ip add dns name="Ethernet" 8.8.4.4 index=2

# Set automatic DNS
netsh interface ip set dns name="Ethernet" dhcp

# Enable/disable network interface
netsh interface set interface "Ethernet" enabled
netsh interface set interface "Ethernet" disabled
```

### Network Configuration Export/Import

```cmd
# Export network configuration
netsh -c interface dump > network_config.txt

# Import network configuration
netsh -f network_config.txt
```

### View Network Connection Status

```cmd
# View TCP connections
netsh interface tcp show global

# View all network configuration
netsh interface ip show config

# View DNS cache
netsh interface ip show dnsservers

# View ARP cache
netsh interface ip show neighbors
```

### WLAN Configuration (Wireless)

```cmd
# View WLAN information
netsh wlan show interfaces

# View saved WLAN profiles
netsh wlan show profiles

# Show profile details (including password)
netsh wlan show profile name="WiFiName" key=clear

# Connect to network
netsh wlan connect name="WiFiName"

# Disconnect
netsh wlan disconnect

# Delete profile
netsh wlan delete profile name="WiFiName"

# Scan available networks
netsh wlan show networks

# Export WLAN configuration
netsh wlan export profile folder=C:\WiFiBackup

# Import WLAN configuration
netsh wlan add profile filename="C:\WiFiBackup\profile.xml"
```

### Network Reset

```cmd
# Reset TCP/IP stack
netsh int ip reset

# Reset Winsock catalog
netsh winsock reset

# Reset firewall configuration
netsh advfirewall reset

# Reset all network settings (requires reboot)
netsh int ip reset
netsh winsock reset
ipconfig /flushdns
ipconfig /release
ipconfig /renew
```

---

## Windows Firewall

### Basic Firewall Management

```cmd
# View firewall status
netsh advfirewall show allprofiles

# Enable firewall (all profiles)
netsh advfirewall set allprofiles state on

# Disable firewall (not recommended)
netsh advfirewall set allprofiles state off

# Enable specific profile firewall
netsh advfirewall set domainprofile state on
netsh advfirewall set privateprofile state on
netsh advfirewall set publicprofile state on

# Restore default firewall settings
netsh advfirewall reset
```

### Firewall Rule Management

```cmd
# View all firewall rules
netsh advfirewall firewall show rule name=all

# View specific rule
netsh advfirewall firewall show rule name="Remote Desktop"

# Allow specific port (inbound)
netsh advfirewall firewall add rule name="Allow TCP 80" dir=in action=allow protocol=TCP localport=80

# Allow specific port (outbound)
netsh advfirewall firewall add rule name="Allow TCP 443" dir=out action=allow protocol=TCP localport=443

# Allow specific program
netsh advfirewall firewall add rule name="Allow MyApp" dir=in action=allow program="C:\Program Files\MyApp\app.exe"

# Allow specific IP address
netsh advfirewall firewall add rule name="Allow Specific IP" dir=in action=allow remoteip=192.168.1.100

# Block specific port
netsh advfirewall firewall add rule name="Block TCP 445" dir=in action=block protocol=TCP localport=445

# Delete rule
netsh advfirewall firewall delete rule name="Allow TCP 80"

# Enable/disable rule
netsh advfirewall firewall set rule name="Remote Desktop" new enable=yes
netsh advfirewall firewall set rule name="Remote Desktop" new enable=no
```

### Firewall Logging

```cmd
# Enable firewall logging
netsh advfirewall set allprofiles logging filename %systemroot%\system32\LogFiles\Firewall\pfirewall.log
netsh advfirewall set allprofiles logging maxfilesize 4096
netsh advfirewall set allprofiles logging droppedconnections enable
netsh advfirewall set allprofiles logging allowedconnections enable

# View logging configuration
netsh advfirewall show allprofiles logging

# View log file
type %systemroot%\system32\LogFiles\Firewall\pfirewall.log
```

---

## PowerShell Network Commands

PowerShell provides more powerful and flexible network management capabilities.

### Basic Network Diagnostics

```powershell
# Test network connection
Test-Connection -ComputerName 192.168.1.1 -Count 4

# Continuous ping
Test-Connection -ComputerName 192.168.1.1 -Continuous

# Test port connectivity
Test-NetConnection -ComputerName 192.168.1.1 -Port 80

# Route trace
Test-NetConnection -ComputerName www.google.com -TraceRoute

# DNS resolution
Resolve-DnsName www.google.com
Resolve-DnsName -Name www.google.com -Type A
Resolve-DnsName -Name google.com -Type MX

# Clear DNS cache
Clear-DnsClientCache

# View DNS cache
Get-DnsClientCache
```

### Network Adapter Management

```powershell
# View all network adapters
Get-NetAdapter

# View enabled adapters
Get-NetAdapter | Where-Object {$_.Status -eq "Up"}

# View adapter details
Get-NetAdapter -Name "Ethernet" | Format-List *

# Enable/disable adapter
Enable-NetAdapter -Name "Ethernet"
Disable-NetAdapter -Name "Ethernet"

# Restart adapter
Restart-NetAdapter -Name "Ethernet"

# View adapter statistics
Get-NetAdapterStatistics
```

### IP Configuration Management

```powershell
# View IP configuration
Get-NetIPConfiguration

# View detailed IP configuration
Get-NetIPConfiguration -Detailed

# View specific adapter IP
Get-NetIPAddress -InterfaceAlias "Ethernet"

# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.100 -PrefixLength 24 -DefaultGateway 192.168.1.1

# Remove IP address
Remove-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.100

# Set DNS servers
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("8.8.8.8","8.8.4.4")

# Set DHCP
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
```

### Route Management

```powershell
# View routing table
Get-NetRoute

# View default route
Get-NetRoute -DestinationPrefix "0.0.0.0/0"

# Add static route
New-NetRoute -DestinationPrefix "10.0.0.0/8" -InterfaceAlias "Ethernet" -NextHop 192.168.1.1

# Delete route
Remove-NetRoute -DestinationPrefix "10.0.0.0/8"
```

### Firewall Management

```powershell
# View firewall status
Get-NetFirewallProfile

# Enable firewall
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# Disable firewall
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# View firewall rules
Get-NetFirewallRule

# View specific rule
Get-NetFirewallRule -DisplayName "Remote Desktop*"

# Create firewall rule
New-NetFirewallRule -DisplayName "Allow TCP 80" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

# Delete rule
Remove-NetFirewallRule -DisplayName "Allow TCP 80"

# Enable/disable rule
Enable-NetFirewallRule -DisplayName "Remote Desktop*"
Disable-NetFirewallRule -DisplayName "Remote Desktop*"
```

### View Network Connections

```powershell
# View TCP connections
Get-NetTCPConnection

# View listening ports
Get-NetTCPConnection -State Listen

# View established connections
Get-NetTCPConnection -State Established

# View specific port
Get-NetTCPConnection -LocalPort 80

# View connections with processes
Get-NetTCPConnection | Select-Object LocalAddress,LocalPort,RemoteAddress,RemotePort,State,@{Name="Process";Expression={(Get-Process -Id $_.OwningProcess).ProcessName}}

# View UDP endpoints
Get-NetUDPEndpoint
```

---

## Network Packet Capture Tools

### Using netsh trace

Windows built-in network packet capture tool.

```cmd
# Start capture
netsh trace start capture=yes tracefile=C:\capture.etl

# Start capture (specify adapter)
netsh trace start capture=yes tracefile=C:\capture.etl captureinterface="Ethernet"

# Capture specific IP traffic
netsh trace start capture=yes tracefile=C:\capture.etl IPv4.Address=192.168.1.100

# Stop capture
netsh trace stop

# View supported scenarios
netsh trace show scenarios

# Capture with specific scenario
netsh trace start scenario=InternetClient capture=yes tracefile=C:\capture.etl

# Convert ETL file (use Microsoft Message Analyzer or Wireshark)
# Use etl2pcapng tool to convert to pcapng format
```

### Using pktmon (Windows 10 1809+)

```cmd
# List all network components
pktmon comp list

# Start capture
pktmon start --etw

# Capture to file
pktmon start --etw -f pktmon.etl

# Add filter (specific IP)
pktmon filter add -i 192.168.1.100

# Add filter (specific port)
pktmon filter add -p 80

# List filters
pktmon filter list

# Stop capture
pktmon stop

# View statistics
pktmon counters

# Reset counters
pktmon reset

# Convert to pcapng format (for Wireshark)
pktmon etl2txt pktmon.etl
pktmon pcapng pktmon.etl -o pktmon.pcapng

# Real-time display
pktmon start -c
```

### Wireshark

Most powerful packet capture tool for Windows (requires separate installation).

**Installation:**
1. Download Wireshark: https://www.wireshark.org/download.html
2. Ensure Npcap or WinPcap is installed during setup

**Common Filters:**

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
arp

# Combined filtering
ip.addr == 192.168.1.100 && tcp.port == 80
tcp.flags.syn == 1 && tcp.flags.ack == 0

# HTTP filtering
http.request.method == "GET"
http.host contains "google"
```

---

## Network Troubleshooting Workflow

### 🔍 Systematic Troubleshooting Steps

```
┌─────────────────────────────────────────────────────────┐
│         Windows Network Troubleshooting Workflow        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Check Network Adapter Status                        │
│     └── Device Manager, Network Connections, LEDs       │
│              ↓                                          │
│  2. Check IP Configuration                              │
│     └── ipconfig /all                                   │
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
│     └── nslookup www.google.com                         │
│              ↓                                          │
│  7. Check Routing                                       │
│     └── tracert / pathping                              │
│              ↓                                          │
│  8. Check Firewall                                      │
│     └── netsh advfirewall show currentprofile           │
│              ↓                                          │
│  9. Check Ports and Services                            │
│     └── netstat -ano                                    │
│              ↓                                          │
│  10. Network Reset (last resort)                        │
│      └── netsh int ip reset + netsh winsock reset      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 📋 Quick Diagnosis Script (PowerShell)

```powershell
# network_diagnostic.ps1
Write-Host "========================================"
Write-Host "Windows Network Diagnostic Script"
Write-Host "========================================"
Write-Host ""

Write-Host "[1] Network Adapter Status" -ForegroundColor Cyan
Get-NetAdapter | Format-Table Name, Status, LinkSpeed, MacAddress
Write-Host ""

Write-Host "[2] IP Configuration" -ForegroundColor Cyan
Get-NetIPConfiguration | Format-Table InterfaceAlias, IPv4Address, IPv4DefaultGateway
Write-Host ""

Write-Host "[3] DNS Servers" -ForegroundColor Cyan
Get-DnsClientServerAddress -AddressFamily IPv4 | Format-Table InterfaceAlias, ServerAddresses
Write-Host ""

Write-Host "[4] Connectivity Tests" -ForegroundColor Cyan
Write-Host "Loopback: " -NoNewline
Test-Connection -ComputerName 127.0.0.1 -Count 2 -Quiet
$gateway = (Get-NetRoute -DestinationPrefix "0.0.0.0/0").NextHop
Write-Host "Gateway: " -NoNewline
Test-Connection -ComputerName $gateway -Count 2 -Quiet
Write-Host "Internet: " -NoNewline
Test-Connection -ComputerName 8.8.8.8 -Count 2 -Quiet
Write-Host ""

Write-Host "[5] DNS Resolution Test" -ForegroundColor Cyan
Resolve-DnsName www.google.com -Type A | Select-Object Name, IPAddress
Write-Host ""

Write-Host "[6] Active TCP Connections" -ForegroundColor Cyan
Get-NetTCPConnection -State Established | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State -First 10
Write-Host ""

Write-Host "========================================" -ForegroundColor Green
Write-Host "Diagnostic Complete"
Write-Host "========================================" -ForegroundColor Green
```

---

<p align="center">
  <a href="README.md">Back to Home</a>
</p>
