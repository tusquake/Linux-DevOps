# Linux Networking Commands (ping, traceroute, telnet, netstat, curl, wget, ARP, Nmap, Route Tables)

> **Who is this for?** A backend developer who knows that "the app talks to the database over the network" but has no idea what to do when it suddenly cannot. This guide gives you the toolbox and, more importantly, a **method** for finding out *where* a connection is breaking. In Topic 1 you learned *what* IPs, DNS, and ports are. Here you learn how to **inspect and test** them.

---

## What is it and Why do we need it?

### What are networking commands?

They are command-line tools that let you **see how a machine is connected to the network and test whether it can reach other machines or services**. Each tool examines a different part of the journey a network request takes:

| Question you are asking | Tool |
|-------------------------|------|
| What is my IP address and are my network interfaces up? | `ip addr`, `ip link` |
| Can I reach that machine at all? | `ping` |
| Which path does my traffic take, and where does it stop? | `traceroute`, `mtr` |
| Is that specific port open and accepting connections? | `telnet`, `nc`, `nmap` |
| Which programs on **my** machine are listening or connected? | `ss`, `netstat`, `lsof` |
| Does that web service answer correctly? | `curl`, `wget` |
| How does my machine find other machines on its local network? | `arp`, `ip neigh` |
| Which machines and ports exist on a network? | `nmap` |
| Where does my machine send traffic for a given destination? | `ip route`, `route`, `netstat -r` |
| Does a name resolve to the right IP? | `dig`, `nslookup` |

> **Analogy: A Mechanic's Diagnostic Kit for Plumbing**
> When water stops coming out of a tap, a plumber doesn't guess. They check the tap, then the pipe behind it, then the main valve, then the water supply. Each tool (pressure gauge, camera, flow meter) checks one segment. Networking commands let you check each segment of the "pipe" between your app and the thing it talks to, so you can say precisely: "The break is **here**."

### Why do we need them?

Nearly every production incident has a network angle: "the app can't reach the database", "the site is slow", "the API call times out", "the new server isn't reachable". Without these tools you can only say "it doesn't work". With them, you can say, "DNS resolves correctly, the route is fine, but port 5432 on the database is blocked by a firewall." That sentence is what gets problems fixed in minutes instead of hours.

---

## Core Concepts

### 1. The Layered Model (Why Troubleshooting Has an Order)

Network communication works in layers. A problem at a lower layer breaks everything above it, so you **troubleshoot from the bottom up**.

| Layer (simplified) | What it does | Tools to test it | Analogy |
|--------------------|--------------|------------------|---------|
| **Link / physical** | Is the interface up and connected? | `ip link`, `ip addr` | Is the phone plugged in? |
| **Local network (MAC / ARP)** | Can I talk to my neighbors on the same network? | `arp`, `ip neigh` | Can I talk to people in my own office? |
| **Internet / IP** | Can packets find the destination address? | `ping`, `traceroute`, `ip route` | Does the postal system know a route to that city? |
| **Name resolution (DNS)** | Does the name turn into the right IP? | `dig`, `nslookup` | Is the phone book entry correct? |
| **Transport (TCP / UDP ports)** | Is the specific port open? | `nc`, `telnet`, `nmap`, `ss` | Is the right extension answering? |
| **Application** | Does the service respond correctly? | `curl`, `wget` | Does the person on the line say something sensible? |

---

### 2. IP Addresses, Subnets, and CIDR

A network is divided into **subnets**. A **CIDR** notation such as `192.168.1.0/24` describes a subnet.

| CIDR | Subnet mask | Number of addresses | Typical use |
|------|-------------|---------------------|-------------|
| `/32` | 255.255.255.255 | 1 | A single host |
| `/24` | 255.255.255.0 | 256 (254 usable) | A typical small network |
| `/16` | 255.255.0.0 | 65,536 | A large private network (for example an AWS VPC) |
| `/8` | 255.0.0.0 | about 16.7 million | Very large range |

The number after the slash is how many bits are **fixed** as the network part. `10.0.1.0/24` means "all addresses from 10.0.1.0 to 10.0.1.255".

> **Analogy:** An IP address is like a phone number with an area code. The subnet tells you which part is the "area code" (the network) and which part is the "local number" (the host). Calls within the same area code stay local; anything else goes through the exchange (the gateway).

---

### 3. Gateways and Routing Tables

Every machine has a **routing table**: a list of rules that say "to reach network X, send the packet to Y".

- If the destination is on the **same subnet**, the packet goes **directly** to the machine.
- If the destination is **elsewhere**, the packet goes to the **default gateway** (usually a router), which forwards it onwards.

```text
default via 192.168.1.1 dev eth0           <-- "for anything else, hand it to 192.168.1.1"
192.168.1.0/24 dev eth0 proto kernel       <-- "this network is directly attached on eth0"
10.10.0.0/16 via 192.168.1.254 dev eth0    <-- "to reach 10.10.x.x, go through 192.168.1.254"
```

The kernel picks the **most specific** matching rule (longest prefix). The **default route** (`0.0.0.0/0`) is the catch-all.

> **Analogy:** The routing table is a set of road signs at a junction: "Local streets: turn left. Anything for the airport: take the highway. **Everything else: go to the main roundabout (the default gateway).**"

---

### 4. ARP: Finding a Neighbor's Hardware Address

IP addresses are logical. On the **local network**, devices actually communicate through hardware (**MAC**) addresses burned into network cards, such as `0a:1b:2c:3d:4e:5f`. **ARP (Address Resolution Protocol)** is how a machine learns "which MAC address owns this IP on my local network?". The result is cached in an **ARP (neighbor) table**.

| | DNS | ARP |
|---|-----|-----|
| Translates | Name into IP | IP into MAC |
| Scope | Internet-wide | Local network only |
| Analogy | Phone book: person to phone number | Shouting in the office: "Who has IP 192.168.1.20?" and the owner replies "I do, my desk is number 17" |

---

### 5. Ports, Listening, and Connection States

A **port** identifies a service on a machine. A service that is **listening** is waiting for incoming connections. Important connection states you will see in `ss` or `netstat`:

| State | Meaning |
|-------|---------|
| `LISTEN` | A service is waiting for connections on this port |
| `ESTABLISHED` | An active connection is open |
| `TIME_WAIT` | A connection recently closed and the system is waiting briefly before reusing the port (normal) |
| `CLOSE_WAIT` | The other side closed, but the local application has not closed its end. Many of these usually mean an application bug (leaked connections) |
| `SYN_SENT` | Trying to connect, with no answer yet (often a firewall dropping packets) |

**Port scan results (Nmap)** use three words: **open** (service answering), **closed** (reachable but nothing listening), **filtered** (a firewall is silently dropping the packets, so no reply).

---

### 6. Binding: `0.0.0.0` vs. `127.0.0.1` (A Classic Gotcha)

When an application starts, it **binds** to an address and port:

| Listening address | Reachable from | Meaning |
|-------------------|----------------|---------|
| `127.0.0.1` (localhost) | Only the **same machine** | "Private to this house" |
| `192.168.1.10` (a specific interface IP) | That network only | "Only this front door" |
| `0.0.0.0` (all interfaces) | **Any network** the machine is on (subject to firewalls) | "All doors open" |

A very common bug: the app works with `curl localhost:3000` on the server, but not from outside, because it listens only on `127.0.0.1`.

> **Analogy:** A shop with a back-office phone line that only works inside the building (`127.0.0.1`) vs. a public phone line that anyone can call (`0.0.0.0`).

---

### 7. A Note on Old vs. New Tools

Many tutorials use the older **net-tools** package (`ifconfig`, `netstat`, `route`, `arp`). It is **deprecated** and often not installed by default. The modern replacement is **iproute2**:

| Old (net-tools) | Modern (iproute2) |
|-----------------|-------------------|
| `ifconfig` | `ip addr`, `ip link` |
| `netstat -tulpn` | `ss -tulpn` |
| `netstat -r`, `route -n` | `ip route` |
| `arp -a` | `ip neigh` |

You still need to **recognize the old commands**, since they appear in interviews, older servers, and countless documents, so both are shown below.

---

## Practical Examples & Commands

> Install the tools used in this guide on Ubuntu/Debian:
> `sudo apt update && sudo apt install -y iputils-ping traceroute mtr-tiny telnet netcat-openbsd net-tools dnsutils nmap curl wget tcpdump arping`

### 1. Inspect Your Own Network Configuration

```bash
ip addr show
# Show every network interface with its IP addresses (short form: ip a).
# Look for "inet 10.0.1.15/24" = your IPv4 address and subnet, "state UP" = interface is active.

ip -br addr
# Brief, one-line-per-interface summary. Easy to read.

ip link show
# Show interfaces at the link layer, including the MAC address ("link/ether ...") and state.

hostname -I
# Print all your IP addresses on one line.

curl -s ifconfig.me
# Find your PUBLIC IP as seen from the Internet (-s = silent mode).

ifconfig
# Older equivalent of "ip addr" (requires net-tools).

sudo ip link set eth0 down && sudo ip link set eth0 up
# Disable and re-enable an interface. Careful: on a remote server this can cut your own SSH session!
```

### 2. `ping`: Is the Machine Reachable?

`ping` sends small **ICMP echo request** packets and waits for **echo replies**, measuring the round-trip time.

```bash
ping -c 4 8.8.8.8
# Send 4 packets to a public IP (Google DNS) and stop.
# Success = replies with "time=12.3 ms" and "0% packet loss".
# This tests raw Internet connectivity WITHOUT needing DNS.

ping -c 4 google.com
# Same, but using a NAME. If this fails while pinging 8.8.8.8 works, the problem is DNS.

ping -c 4 -i 0.5 10.0.1.20
# -i 0.5 = wait half a second between packets.

ping -c 3 -s 1400 10.0.1.20
# -s sets the packet size in bytes. Useful for finding MTU (maximum packet size) problems.

ping -c 3 -W 2 10.0.1.20
# -W 2 = wait at most 2 seconds for each reply.

ping6 -c 3 ipv6.google.com
# Ping an IPv6 destination (or: ping -6 ...).
```

Reading the result:

```text
64 bytes from 142.250.190.78: icmp_seq=1 ttl=117 time=12.4 ms
|                                          |       |
|                                          |       +-- round-trip time (latency)
|                                          +-- TTL: remaining "hop budget" of the reply
+-- bytes received
...
4 packets transmitted, 4 received, 0% packet loss
```

> **Important:** A failed `ping` does **not** always mean the machine is down. Many firewalls and cloud security groups **block ICMP** by default (AWS, for example, requires you to explicitly allow it). Always follow up with a **port test** (`nc` or `curl`).
>
> **Analogy:** `ping` is shouting "Hello?" into a tunnel and timing the echo. No echo might mean nobody is there, or that they've chosen not to reply to shouting.

### 3. `traceroute` and `mtr`: Where Does the Path Break?

`traceroute` reveals each **hop** (router) between you and the destination. It works by sending packets with a **TTL (Time To Live)** of 1, then 2, then 3, and so on. Each router reduces the TTL by 1; the router that makes it zero replies "time exceeded", which reveals its address.

```bash
traceroute google.com
# Show each router on the path. Each line = one hop, with three round-trip times.

traceroute -n google.com
# -n = do not resolve names (faster and avoids DNS noise).

traceroute -I google.com
# Use ICMP packets instead of the default UDP (works better through some firewalls; may need sudo).

sudo traceroute -T -p 443 example.com
# Use TCP packets to port 443. Useful when ICMP and UDP are filtered but HTTPS is allowed.

tracepath google.com
# Similar, but needs no root privileges, and also discovers the MTU.

mtr -rwc 10 google.com
# mtr = ping + traceroute combined, continuously. In report mode:
#   -r report, -w wide output, -c 10 send 10 packets per hop.
# The "Loss%" column helps you spot exactly which hop drops packets.
```

How to read `traceroute`:

```text
 1  192.168.1.1     1.2 ms   1.1 ms   1.0 ms     <-- your router
 2  10.20.0.1       5.4 ms   5.3 ms   5.6 ms     <-- your ISP
 3  * * *                                       <-- this hop did not reply (often just ignoring probes)
 4  142.250.190.78  12.4 ms  12.3 ms  12.5 ms    <-- destination reached
```

`* * *` on a **middle** hop is usually harmless (the router just doesn't answer probes). If **every hop after a certain point** is `* * *` and the destination is never reached, the path is blocked at or just after that point.

### 4. `telnet` and `nc` (netcat): Is That Port Open?

The most useful thing `telnet` does today is **test whether a TCP port accepts connections**. (Never use it for remote login: it sends everything, including passwords, in plain text. Use SSH.)

```bash
telnet 10.0.1.20 5432
# Try to open a TCP connection to port 5432 (PostgreSQL) on 10.0.1.20.
#   "Connected to 10.0.1.20."        -> the port is OPEN and reachable
#   "Connection refused"             -> the machine is reachable, but nothing listens there
#   hangs, then "Connection timed out" -> a firewall is silently dropping packets
# To exit once connected: press Ctrl + ], then type "quit" and press Enter.

telnet example.com 80
# Once connected, you can even type a raw HTTP request by hand:
#   GET / HTTP/1.1
#   Host: example.com
#   (then press Enter twice)
```

`nc` (netcat) is the modern, scriptable "Swiss army knife" for the same job:

```bash
nc -zv 10.0.1.20 5432
# -z = just test the connection, don't send data
# -v = verbose, report the result ("succeeded!" or "Connection refused")

nc -zv -w 3 example.com 443
# -w 3 = give up after 3 seconds (avoids long hangs).

nc -zv 10.0.1.20 20-25
# Scan a RANGE of ports (20 to 25).

nc -l 8080
# LISTEN on port 8080 on this machine (like a tiny server).
# On another machine, run:  nc <this-server-ip> 8080   and type text: it appears here.
# Useful for testing whether a firewall lets traffic through to a given port.
```

> **Analogy:** `telnet host port` is **knocking on a specific door** and noting what happens: someone opens ("Connected"), a sign says "nobody lives here" ("refused"), or the door is bricked up and nothing happens at all ("timeout").

### 5. `ss` and `netstat`: What Is My Machine Listening On or Connected To?

```bash
ss -tulpn
# Show listening sockets with owning programs.
#   -t TCP   -u UDP   -l listening only   -p show the process   -n numeric (no name lookups)
# Use sudo to see the process names for all users.

ss -tulpn | grep :80
# Is anything listening on port 80? Which program?

ss -tan
# Show ALL TCP connections (-a all, -n numeric), including their state.

ss -tan state established
# Only established TCP connections.

ss -s
# A summary: totals of connections per state.

ss -tan | grep -c ESTAB
# Count the established connections.

netstat -tulpn
# The old equivalent of "ss -tulpn" (requires net-tools; use sudo for process names).

netstat -an | grep LISTEN
# Show all listening ports.

netstat -i
# Show network interface statistics (packets and errors).

netstat -s
# Per-protocol statistics. Retransmissions and errors hint at network trouble.

sudo lsof -i :3000
# Another way to ask "who is using port 3000?" (shows the process and user).
```

Reading `ss -tulpn` output:

```text
Netid State  Local Address:Port   Peer Address:Port  Process
tcp   LISTEN 0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=812,fd=3))
tcp   LISTEN 127.0.0.1:5432       0.0.0.0:*          users:(("postgres",pid=990,fd=5))
tcp   LISTEN *:3000               *:*                users:(("node",pid=1201,fd=18))
```

- `sshd` listens on `0.0.0.0:22`: reachable from anywhere (subject to firewalls).
- `postgres` listens on `127.0.0.1:5432`: **only this machine** can connect. A remote app server will fail until the database is configured to listen on a reachable address.
- `node` listens on `*:3000`: all interfaces.

### 6. `curl`: Talk HTTP from the Command Line

`curl` is a full HTTP client: it is the single most useful tool for testing web services and APIs.

```bash
curl https://example.com
# Fetch a page and print the body.

curl -I https://example.com
# HEAD request: show only the response headers (status code, server, content type, redirects).

curl -v https://example.com
# Verbose: shows DNS, TCP connect, TLS handshake, and request/response headers. Great for debugging.

curl -s -o /dev/null -w "%{http_code}\n" https://example.com
# Print ONLY the HTTP status code.
#   -s silent   -o /dev/null discard the body   -w custom output format

curl -L http://example.com
# -L follows redirects (for example HTTP to HTTPS).

curl -o page.html https://example.com
# Save the output to a file with a chosen name.

curl -O https://example.com/files/data.zip
# -O saves the file using its original remote name (data.zip).

curl --max-time 5 --connect-timeout 3 https://example.com
# Do not wait forever: give up connecting after 3 s, and the whole request after 5 s.

curl -fsS https://example.com/health || echo "UNHEALTHY"
# -f fail on HTTP errors (4xx/5xx), -s quiet, -S still show errors.
# Perfect for health checks in scripts and cron jobs (the exit code reflects success).

curl --retry 3 --retry-delay 2 https://example.com/api
# Automatically retry failures.
```

Calling APIs:

```bash
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"alice","role":"dev"}'
# -X POST      HTTP method
# -H           add a request header (content type, auth token)
# -d           the request body (JSON data)

curl -u alice:password https://example.com/protected
# Basic authentication (avoid typing real passwords on a shared command line: they land in history).

curl -k https://self-signed.internal
# -k skips TLS certificate verification. For TESTING only. Never in production scripts.
```

Advanced debugging:

```bash
curl --resolve example.com:443:203.0.113.25 https://example.com
# Force a name to resolve to a specific IP FOR THIS REQUEST ONLY.
# Perfect for testing a new server BEFORE changing DNS.

curl -H "Host: example.com" http://203.0.113.25/
# Send a request to an IP but present a specific Host header (virtual hosts).

curl -s -o /dev/null -w "dns:%{time_namelookup}s connect:%{time_connect}s tls:%{time_appconnect}s ttfb:%{time_starttransfer}s total:%{time_total}s\n" https://example.com
# Break down WHERE the time is spent: DNS, TCP connect, TLS, time to first byte, total.
# Slow DNS? Slow server? This tells you.

curl -x http://proxy.corp:3128 https://example.com
# Send the request through a proxy.
```

### 7. `wget`: Download Files

```bash
wget https://example.com/files/data.zip
# Download a file into the current directory.

wget -O latest.zip https://example.com/files/data.zip
# Save under a different name (capital -O).

wget -P ~/downloads https://example.com/files/data.zip
# Save into a specific directory.

wget -c https://example.com/files/big.iso
# -c = continue a partially downloaded file (resume after an interruption).

wget -q https://example.com/file          # Quiet mode, no output
wget -b https://example.com/big.iso       # -b = download in the BACKGROUND (log in wget-log)
wget --limit-rate=1m https://example.com/big.iso     # Cap the speed at 1 MB/s
wget --tries=5 --timeout=10 https://example.com/file # Retry up to 5 times, 10 s timeout
wget -i urls.txt                          # Download every URL listed in a file

wget --spider https://example.com/page
# Check whether a URL exists WITHOUT downloading it.

wget --mirror --convert-links --no-parent https://example.com/docs/
# Make a local copy of a website section. Use responsibly.
```

**`curl` vs. `wget` at a glance:**

| | `curl` | `wget` |
|---|--------|--------|
| Best at | Testing APIs and HTTP details; many protocols | Downloading files and mirroring sites |
| Output default | Prints to the screen | Saves to a file |
| Resume downloads | `-C -` | `-c` |
| Recursive download | No | Yes (`-r`, `--mirror`) |
| Custom methods, headers, JSON | Excellent | Limited |

### 8. DNS Checks (Name Resolution)

```bash
dig example.com +short
# Print only the IP address(es) for the name.

dig example.com
# Full answer: query time, DNS server used, TTL of the record, and more.

dig @8.8.8.8 example.com +short
# Ask a SPECIFIC DNS server (Google's) instead of your default one.
# If this works but the default does not, your configured DNS server is the problem.

dig -x 8.8.8.8 +short
# Reverse lookup: IP back to name.

nslookup example.com
# An older, simpler lookup tool, still common.

getent hosts example.com
# Resolve the way your applications do (honours /etc/hosts and the system resolver).

cat /etc/resolv.conf
# Which DNS servers does this machine use? (On Ubuntu this may point to a local resolver such as 127.0.0.53.)

resolvectl status
# Detailed DNS configuration on systemd-based systems.

cat /etc/hosts
# Local name-to-IP overrides, checked BEFORE DNS. Example line:  10.0.1.20  db.internal
```

### 9. ARP: Neighbors on the Local Network

```bash
ip neigh show
# Show the neighbor (ARP) table: IP address, MAC address, and state (REACHABLE, STALE, FAILED).

arp -a
# The older equivalent (net-tools). Format: hostname (IP) at MAC [ether] on interface.

arp -n
# Numeric output, without DNS lookups (faster).

sudo arping -c 3 -I eth0 10.0.1.20
# Send ARP requests (not ICMP) to an address on the SAME network.
# Works even when ICMP ping is blocked. Ideal for checking a local machine's presence.

sudo arping -D -I eth0 -c 2 10.0.1.50
# Duplicate Address Detection: exits with a failure code if some other device already uses this IP.

sudo ip neigh flush dev eth0
# Clear the ARP cache for an interface (forces fresh lookups).

sudo ip neigh add 10.0.1.99 lladdr 0a:1b:2c:3d:4e:5f dev eth0
# Manually add a static ARP entry (rarely needed).
```

Typical use of ARP in troubleshooting:

- A `FAILED` or `INCOMPLETE` entry for a local IP means that machine is not answering ARP: it is down, unplugged, or on a different network segment.
- **Two different MAC addresses** claiming the same IP means an **IP conflict**: a classic cause of "it works, then it doesn't".

### 10. Nmap: Discover Hosts and Open Ports

> **Legal and ethical warning:** Only scan machines and networks that **you own or have explicit written permission to test**. Scanning others can violate laws, cloud-provider policies, and company rules, and can trigger intrusion alerts. Also check your cloud provider's policy before scanning instances in the cloud.

```bash
sudo apt install -y nmap          # Install

nmap 10.0.1.20
# Scan the 1000 most common ports on one host.

nmap -p 22,80,443 10.0.1.20
# Scan only specific ports.

nmap -p 1-1024 10.0.1.20          # A range of ports
nmap -p- 10.0.1.20                # ALL 65535 ports (slow)

nmap -sV 10.0.1.20
# Version detection: which software and version is behind each open port?
# (For example "OpenSSH 8.9" or "nginx 1.24".)

sudo nmap -O 10.0.1.20
# Try to guess the operating system (needs root).

nmap -sn 10.0.1.0/24
# PING SWEEP: list which hosts are alive in the subnet, without scanning their ports.

nmap -Pn 10.0.1.20
# Skip the "is it up?" check. Use when a host blocks ping but may still have open ports.

sudo nmap -sS 10.0.1.20
# SYN ("stealth") scan: the default for root; fast and does not complete the TCP handshake.

nmap -sT 10.0.1.20
# Full TCP connect scan: the default without root.

nmap -sU -p 53,123 10.0.1.20
# Scan UDP ports (slower and less reliable; needs root).

nmap -T4 --open 10.0.1.20
# -T4 faster timing, --open shows only open ports.

nmap -A 10.0.1.20
# "Aggressive": OS detection, version detection, and default scripts.

nmap -oN scan-result.txt 10.0.1.0/24
# Save the results to a text file (-oN normal format).
```

Reading the output:

```text
PORT     STATE    SERVICE
22/tcp   open     ssh        <-- a service answers: reachable
80/tcp   open     http
443/tcp  closed   https      <-- host replied "nothing here": no service on this port
5432/tcp filtered postgresql <-- no reply at all: a firewall is dropping packets
```

Use Nmap to **audit your own servers**: any port that is `open` but shouldn't be (for example a database exposed to the Internet) is a security finding.

### 11. Routing Tables: Where Does My Traffic Go?

```bash
ip route show
# Show the routing table (short form: ip r).
# Example output:
#   default via 10.0.1.1 dev eth0 proto dhcp metric 100
#   10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.15

route -n
# The older equivalent (net-tools). -n = numeric. Column "Gateway" 0.0.0.0 means "directly attached".

netstat -rn
# Another old way to print the routing table.

ip route get 8.8.8.8
# Ask the kernel: "WHICH route and interface would you use to reach 8.8.8.8?"
# Excellent for confirming that traffic leaves through the interface you expect.

ip route get 10.20.5.9
# Same, for an internal address.

sudo ip route add 10.20.0.0/16 via 10.0.1.254 dev eth0
# Add a ROUTE: "to reach 10.20.x.x, send traffic via the gateway 10.0.1.254".

sudo ip route del 10.20.0.0/16
# Delete that route.

sudo ip route add default via 10.0.1.1
# Set the DEFAULT gateway (the catch-all route).

sudo route add -net 10.20.0.0 netmask 255.255.0.0 gw 10.0.1.254
# The older syntax for the same "add route" operation.
```

> **Note:** Routes added with these commands are **temporary**; they vanish on reboot. To make them permanent, configure them in the system's network configuration (on Ubuntu: **Netplan** files in `/etc/netplan/`, then `sudo netplan apply`; on Red Hat-family systems: NetworkManager with `nmcli`). In cloud environments, remember that **cloud route tables** (such as AWS VPC route tables) are a separate layer controlled in the cloud console.

### 12. Capture Packets with `tcpdump` (Bonus)

```bash
sudo tcpdump -i eth0 -n port 80 -c 10
# Capture 10 packets on interface eth0 involving port 80.
#   -i eth0 which interface   -n no name lookups   -c 10 stop after 10 packets

sudo tcpdump -i any -n host 10.0.1.20 and port 5432
# Watch traffic between this machine and the database.
# If you see packets LEAVING but no replies coming back, a firewall is dropping them.

sudo tcpdump -i eth0 -w capture.pcap port 443
# Save to a file for later analysis (for example in Wireshark).
```

### 13. A Systematic Troubleshooting Checklist

```bash
# Problem: "My app server cannot reach the database db.internal:5432"

ip -br addr                          # 1. Do I have an IP and is the interface UP?
ip route                             # 2. Do I have a default route / a route to the DB network?
ping -c 2 10.0.1.1                   # 3. Can I reach my gateway?
getent hosts db.internal             # 4. Does the name resolve? To the RIGHT IP?
dig db.internal +short               #    (compare with the system resolver result)
ping -c 3 10.0.2.8                   # 5. Can I reach the DB machine by IP? (ICMP may be blocked)
traceroute -n 10.0.2.8               # 6. Where does the path stop?
nc -zv -w 3 10.0.2.8 5432            # 7. Is the DB PORT reachable?
#    refused  -> machine reachable, but DB not listening (or listening on 127.0.0.1 only)
#    timeout  -> firewall / security group / network ACL dropping packets
# On the DB server itself:
sudo ss -tulpn | grep 5432           # 8. Is the DB actually listening, and on which address?
sudo tcpdump -i any -n port 5432     # 9. Do packets arrive at all?
```

---

## Real-World Use Case

### Scenario: "The web app can't connect to the database", diagnosed layer by layer

An engineer deploys a new version of a Node.js API on `app-01` (private IP `10.0.1.15`). The logs say `ECONNREFUSED 10.0.2.8:5432` (or `ETIMEDOUT`). The database `db-01` (private IP `10.0.2.8`) runs PostgreSQL in a different subnet.

```
   Internet
      |
  +---v----------------+          +--------------------+
  | app-01  10.0.1.15  | -------> | db-01   10.0.2.8   |
  | Node.js API :3000  |  :5432   | PostgreSQL :5432   |
  +--------------------+          +--------------------+
     Subnet 10.0.1.0/24             Subnet 10.0.2.0/24
```

**Investigation (using the commands above):**

| Step | Command (run on app-01 unless noted) | Result | Conclusion |
|------|--------------------------------------|--------|------------|
| 1 | `ip -br addr` | `eth0 UP 10.0.1.15/24` | Interface and IP fine |
| 2 | `ip route` | default via 10.0.1.1 | Routing configured |
| 3 | `getent hosts db.internal` | `10.0.2.8 db.internal` | DNS correct |
| 4 | `ping -c 3 10.0.2.8` | replies, 0% loss | Network path works |
| 5 | `nc -zv -w 3 10.0.2.8 5432` | **Connection timed out** | Port blocked or not reachable |
| 6 | (on db-01) `sudo ss -tulpn \| grep 5432` | `127.0.0.1:5432` | **PostgreSQL only listens on localhost!** |

**Root cause found:** the database only binds to `127.0.0.1`, so remote machines cannot connect, even though the network path is perfect.

**Fix:**

```bash
# On db-01: allow PostgreSQL to listen on the private interface (config path varies by version)
sudo nano /etc/postgresql/16/main/postgresql.conf
#   listen_addresses = 'localhost,10.0.2.8'     # was: 'localhost'

sudo nano /etc/postgresql/16/main/pg_hba.conf
#   host  appdb  appuser  10.0.1.0/24  scram-sha-256   # allow only the app subnet

sudo systemctl restart postgresql
sudo ss -tulpn | grep 5432             # Now shows 10.0.2.8:5432
```

After this, `nc -zv 10.0.2.8 5432` from app-01 reports `succeeded!`. If it **still** timed out after the database was listening correctly, the next suspects would be the **cloud security group** (is port 5432 allowed from `10.0.1.0/24`?), a **network ACL**, or the OS firewall (`sudo ufw status`). The method stays the same: test each layer and move on only when it passes.

**Why this matters in production:**

- You turn a vague complaint into a **precise diagnosis** in about five minutes.
- You know whether to call the **network team**, the **cloud admin**, the **DBA**, or the **developer**.
- `refused` and `timeout` mean different things: `refused` = the machine said "no" (nobody listening), `timeout` = the packets vanished (firewall or routing).
- You can confirm the fix with objective evidence rather than hope.

> **Analogy:** A courier cannot deliver a parcel. You check: is the truck running (interface), does the driver have a map (routes), is the address right (DNS), is the street reachable (ping/traceroute), is the front door answering (port), and is the recipient actually home and accepting parcels (application)? The break is wherever the first "no" appears.

---

## Interview Questions & Answers

### Q1. `ping` to a server works, but the website does not load. How do you troubleshoot?

**Answer:**
A successful ping only proves **IP-level connectivity** over ICMP. I would continue up the stack: (1) check **DNS** with `dig example.com +short` to confirm the name resolves to the correct IP; (2) test the **port** with `nc -zv host 443` or `telnet host 443` to see if it's open, refused, or timing out; (3) test the **application** with `curl -v https://host` to view the TLS handshake, status code, and headers; (4) on the server, confirm the service is **listening** with `ss -tulpn | grep :443`, and check that it is bound to the right address (not only `127.0.0.1`); (5) check **firewalls and security groups** (`ufw status`, cloud rules) and the service logs. "Refused" usually means nothing is listening; "timeout" usually means a firewall is dropping packets.

---

### Q2. How do you find out which process is using port 8080, and what is the difference between `netstat` and `ss`?

**Answer:**
I run `sudo ss -tulpn | grep :8080` (or `sudo lsof -i :8080`, or the older `sudo netstat -tulpn | grep 8080`). The flags are `-t` TCP, `-u` UDP, `-l` listening, `-p` process, `-n` numeric. `netstat` is from the older **net-tools** package and is deprecated; `ss` is part of **iproute2**, faster on busy systems because it reads information directly from the kernel, and has similar syntax. Both show the same kind of data, so I know both, but I prefer `ss` on modern systems.

---

### Q3. An application works with `curl localhost:3000` on the server but is unreachable from other machines. What could be wrong?

**Answer:**
The most common cause is that the application is **bound to `127.0.0.1`** (loopback) instead of `0.0.0.0` (all interfaces), so it only accepts local connections. I'd confirm with `ss -tulpn | grep 3000` and look at the **local address column**. The fix is to change the app's host/bind setting to `0.0.0.0` (or the private interface IP). If it is already bound correctly, the next suspects are the **OS firewall** (`ufw`, `iptables`), the **cloud security group or network ACL**, and routing. I'd verify from the client with `nc -zv server-ip 3000` to differentiate "refused" from "timeout".

---

### Q4. How does `traceroute` work, and what does `* * *` mean?

**Answer:**
`traceroute` sends packets with an incrementing **TTL (Time To Live)**: first TTL=1, then 2, then 3, and so on. Each router along the path decreases the TTL by one; the router that reduces it to zero discards the packet and sends back an ICMP "Time Exceeded" message, revealing its address and the round-trip time. Repeating this builds a hop-by-hop map to the destination. A line of `* * *` means no reply was received for that hop within the timeout: often a router that simply does not answer probes (harmless if later hops respond), but if all hops after a certain point show `* * *` and the destination is never reached, traffic is probably being blocked there. `mtr` combines `traceroute` and `ping` to show packet loss per hop.

---

### Q5. What is ARP, how is it different from DNS, and how do you inspect it?

**Answer:**
**ARP (Address Resolution Protocol)** maps an **IP address to a MAC (hardware) address** on the **local network segment**; a device broadcasts "who has IP x.x.x.x?" and the owner replies with its MAC address. **DNS** maps a **name to an IP address** and works across the Internet. I inspect the ARP cache with `ip neigh show` (or `arp -a`), and can probe a local host with `arping`, which works even if ICMP is blocked. In troubleshooting, a neighbor entry stuck in `FAILED` or `INCOMPLETE` means the local host isn't answering, while two different MAC addresses for the same IP point to an **IP address conflict**.

---

## Quick Recap Cheat Sheet

| Task | Command |
|------|---------|
| My IPs and interfaces | `ip -br addr`, `ip link`, `hostname -I` |
| My public IP | `curl -s ifconfig.me` |
| Is a host reachable? | `ping -c 4 host` |
| Path to a host | `traceroute -n host`, `mtr -rwc 10 host` |
| Is a TCP port open? | `nc -zv -w 3 host port`, `telnet host port` |
| What is listening here? | `sudo ss -tulpn`, `netstat -tulpn` |
| Who uses port N? | `sudo lsof -i :N`, `ss -tulpn \| grep :N` |
| Connection summary | `ss -s`, `ss -tan state established` |
| HTTP status only | `curl -s -o /dev/null -w "%{http_code}\n" URL` |
| Headers and TLS details | `curl -I URL`, `curl -v URL` |
| Timing breakdown | `curl -w "dns:%{time_namelookup} total:%{time_total}\n" -o /dev/null -s URL` |
| Test before DNS change | `curl --resolve host:443:IP https://host` |
| Download / resume | `wget URL`, `wget -c URL` |
| DNS lookup | `dig name +short`, `getent hosts name` |
| ARP / neighbors | `ip neigh show`, `arp -a`, `arping` |
| Find hosts in a subnet | `nmap -sn 10.0.1.0/24` |
| Scan ports and versions | `nmap -p 22,80,443 -sV host` |
| Routing table | `ip route`, `route -n`, `netstat -rn` |
| Which route is used? | `ip route get 8.8.8.8` |
| Add / delete route | `sudo ip route add NET via GW`, `sudo ip route del NET` |
| Capture packets | `sudo tcpdump -i eth0 -n port 80` |

**Result meanings:**
- **Connection refused:** machine reachable, nothing listening on that port.
- **Timeout / filtered:** packets silently dropped (firewall, security group, routing).
- **Listening on `127.0.0.1`:** reachable only from the same machine.
- **ICMP ping blocked is common:** always follow up with a port test.
