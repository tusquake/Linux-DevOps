# Internet and Servers Basics

> **Who is this for?** A backend developer who writes code (APIs, databases, business logic) but has never had to think about *where* that code actually runs or *how* a user's browser reaches it. No prior operations knowledge is assumed.

---

## What is it and Why do we need it?

### The Internet

The **Internet** is a giant, worldwide network of computers that are all connected and agree to talk to each other using the same set of rules.

> **Analogy: The Global Postal System**
> Imagine every house on Earth has a unique address. You can write a letter, put it in an envelope, and the postal system figures out which trucks, planes, and sorting centers to use to deliver it. You don't need to know the route. You only need the destination address and a standard envelope format.
> The Internet works the same way: computers have addresses, data travels in standard "envelopes" (packets), and routers act as sorting centers that decide where each packet goes next.

### Servers

A **server** is just a computer whose job is to *wait for requests* and *respond to them*. It is not magic. It is the same kind of machine as your laptop, but it usually has no screen or keyboard, runs 24/7, and runs software that listens for incoming connections.

> **Analogy: A Restaurant Kitchen**
> You (the **client**) sit at a table and place an order. The kitchen (the **server**) is always open, receives orders, prepares the food, and sends it back. You never go into the kitchen, and the kitchen never leaves the restaurant to find you. It simply waits for orders.

### Why do we need to understand this?

When you write backend code, it works perfectly on your laptop (`localhost:3000`). But real users are not sitting at your laptop. To make your app available to the world, you need to:

1. Put your code on a computer that is **always on** (a server).
2. Give that computer an **address** people can find (IP address / domain name).
3. Make sure it is **reachable and secure** (ports, firewalls).

As a DevOps engineer, you are responsible for everything between "the code is written" and "the user sees the page". You cannot do that if you don't understand how that journey works.

---

## Core Concepts

### 1. Client-Server Model

Almost everything on the Internet follows one pattern:

| Role | What it does | Examples |
|------|--------------|----------|
| **Client** | Starts the conversation by sending a **request** | Browser, mobile app, `curl`, Postman |
| **Server** | Listens and sends back a **response** | Web server, database server, API server |

> **Analogy:** Customer = client. Kitchen = server. The order slip = request. The plate of food = response.

---

### 2. IP Address: "Where is the computer?"

An **IP address** is a unique number that identifies a device on a network.

- **IPv4**: four numbers from 0–255, e.g. `192.168.1.10` (about 4.3 billion possible addresses, and we have mostly run out).
- **IPv6**: a much longer format, e.g. `2001:0db8:85a3::8a2e:0370:7334` (created because IPv4 ran out).

**Public vs. Private IPs**

| Type | Visible from Internet? | Example ranges | Used for |
|------|------------------------|----------------|----------|
| **Public IP** | Yes | Anything not in private ranges, e.g. `54.210.12.7` | Servers that users must reach |
| **Private IP** | No (only inside your own network) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Your home Wi-Fi, internal company machines, databases |

> **Analogy: Office Building**
> The building's **street address** (public IP) is what outsiders use to find you. Inside, each desk has an **extension number** (private IP). Visitors can't dial an extension directly from the street. They have to go through the front desk first.

---

### 3. DNS: "What is the address for this name?"

Humans remember `google.com`. Computers need `142.250.190.78`. **DNS (Domain Name System)** is the service that translates names into IP addresses.

> **Analogy: The Phone Book / Contacts App**
> You tap "Mom" and your phone dials the number. You never memorize the digits. DNS is the contacts app of the Internet.

**Common DNS record types** (you'll see these constantly):

| Record | Meaning | Analogy |
|--------|---------|---------|
| **A** | Name → IPv4 address | "Mom" → 555-0101 |
| **AAAA** | Name → IPv6 address | Same, but a longer number format |
| **CNAME** | Name → another name (alias) | "Mum" is just another name for "Mom" |
| **MX** | Where to deliver email for the domain | Mail forwarding address |
| **NS** | Which servers are the authority for this domain | Which phone book is the official one |
| **TXT** | Free-form text, often used for verification | A sticky note attached to the entry |

---

### 4. How a Request Travels: Step by Step

What happens when you type `https://example.com` into a browser?

```
 You (Browser)
     │
     │ 1. "What's the IP of example.com?"
     ▼
 DNS Resolver ──► (asks Root → TLD → Authoritative DNS servers if it doesn't know)
     │
     │ 2. "It's 93.184.216.34"
     ▼
 You (Browser)
     │
     │ 3. TCP handshake (establish a connection)
     │ 4. TLS handshake (set up encryption for HTTPS)
     │ 5. Send HTTP request: GET / HTTP/1.1
     ▼
 Server (93.184.216.34)
     │
     │ 6. Web server software processes it, maybe talks to a database
     │ 7. Sends back HTTP response: 200 OK + HTML
     ▼
 You (Browser renders the page)
```

> **Analogy: Sending a Registered Letter**
> 1. Look up the person's address in the phone book (**DNS**).
> 2. Call ahead to confirm they're home and will accept the letter (**TCP handshake**).
> 3. Put the letter in a locked, tamper-proof envelope (**TLS/HTTPS**).
> 4. Hand it over with your question inside (**HTTP request**).
> 5. They write back and send a reply (**HTTP response**).

---

### 5. Protocols: "The Agreed-Upon Rules"

A **protocol** is simply a set of rules two computers follow so they understand each other.

| Protocol | Purpose | Default Port | Analogy |
|----------|---------|--------------|---------|
| **TCP** | Reliable delivery: guarantees data arrives complete and in order | n/a (transport layer) | Courier that requires a signature and re-sends lost parcels |
| **UDP** | Fast delivery, no guarantees | n/a (transport layer) | Dropping flyers in mailboxes, with no confirmation they arrived |
| **HTTP** | Web pages and APIs (unencrypted) | 80 | Speaking in a crowded room, so anyone can overhear |
| **HTTPS** | HTTP + encryption (TLS) | 443 | Speaking in a private, soundproof room |
| **SSH** | Securely log into and control a remote server | 22 | A secure, private phone line straight to the server's terminal |
| **FTP/SFTP** | Transfer files | 21 / 22 | A delivery truck for files |
| **SMTP** | Sending email | 25 / 587 | Outgoing mail truck |
| **DNS** | Name lookups | 53 | The phone book service |

---

### 6. Ports: "Which door do I knock on?"

An IP address finds the *machine*. A **port** (0–65535) finds the *specific program* on that machine.

> **Analogy: A Hotel**
> The hotel's street address is the **IP address**. Each room number is a **port**. The guest you want (your web server, your database, your SSH service) lives in a specific room. Port `443` is the "web guest", port `22` is the "admin guest", and port `5432` is the "PostgreSQL guest".

**Well-known ports to memorize:**

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 27017 | MongoDB |

---

### 7. What Exactly Is a Server?

A server has **two meanings**, and people mix them up:

1. **Hardware:** a physical machine in a data center (often a rack-mounted computer with no monitor).
2. **Software:** a *program* that listens for requests (e.g., Nginx is a "web server", PostgreSQL is a "database server").

One physical machine can run many "servers" (software) at the same time, each on a different port.

**Common types of servers:**

| Server Type | Job | Example Software |
|-------------|-----|------------------|
| Web server | Serves web pages / acts as a front door | Nginx, Apache |
| Application server | Runs your backend code | Node.js, Gunicorn, Tomcat |
| Database server | Stores and retrieves data | PostgreSQL, MySQL, MongoDB |
| File server | Stores shared files | NFS, Samba |
| Mail server | Sends/receives email | Postfix |
| DNS server | Answers name lookups | BIND |

---

### 8. Where Do Servers Live? (Physical vs. Virtual vs. Cloud)

| Option | What it is | Analogy |
|--------|-----------|---------|
| **Bare-metal / On-premises** | You buy and own the physical machine | **Buying a house**: full control, but you fix the plumbing |
| **Virtual Machine (VM)** | One physical machine is split into several isolated "virtual computers" | **An apartment building**: many tenants share one building, each with their own locked apartment |
| **Cloud server** (AWS EC2, Azure VM, GCP Compute Engine) | You rent a VM from a provider by the hour | **Renting a hotel room**: pay only while you use it, no maintenance |
| **Container** (Docker) | A lightweight, isolated package for one app | **A shipping container**: standard box that runs the same anywhere |

> You'll use mostly **cloud VMs running Linux** throughout this course. This is why the next topic is Linux itself.

---

### 9. Key Server Qualities DevOps Engineers Care About

| Quality | Meaning | Analogy |
|---------|---------|---------|
| **Uptime / Availability** | Percentage of time the server is working | A shop's opening hours. "99.9%" means it's closed ~8.7 hours a year |
| **Latency** | Delay between request and response | How long a waiter takes to bring your food |
| **Bandwidth** | How much data can flow at once | Width of a highway |
| **Scalability** | Ability to handle more users | Adding more checkout counters when the shop gets busy |
| **Redundancy** | Having backups ready to take over | A spare tire in the trunk |

---

### 10. Firewalls and Security Basics

A **firewall** is a set of rules deciding which traffic is allowed in or out of a server.

> **Analogy: Airport Security**
> Only people with the right ticket (allowed port, allowed IP) pass through. Everyone else is turned back.

**Golden rule:** *Open only the ports you actually need.* A public web server normally exposes only ports **80 and 443** (and **22** for admins, ideally restricted to your own IP). Your database should **never** be open to the whole Internet.

---

## Practical Examples & Commands

> These work on Linux and macOS terminals. On Windows, use WSL (Windows Subsystem for Linux) or Git Bash.

### 1. Find your own IP addresses

```bash
# Show all network interfaces and their private IP addresses
ip addr show
# (older systems/macOS: ifconfig)

# Find your PUBLIC IP address (what the outside world sees)
curl ifconfig.me
# curl = "Client URL" - sends a request to a website and prints the response
```

### 2. Check if a server is reachable: `ping`

```bash
# Send 4 small test packets to Google and wait for replies
ping -c 4 google.com
# -c 4  -> stop after 4 packets (otherwise it runs forever on Linux)
#
# Look for:
#   time=12.3 ms  -> latency (lower is better)
#   0% packet loss -> healthy connection
```

> `ping` is like shouting "Hello, anyone there?" and timing how fast the echo comes back.

### 3. Look up DNS records

```bash
# Simple lookup: name -> IP address
nslookup example.com

# More detailed lookup using "dig" (Domain Information Groper)
dig example.com A +short
# A      -> ask for the IPv4 address record
# +short -> print only the answer, no extra noise

# Look up mail servers for a domain
dig gmail.com MX +short

# Look up the authoritative name servers
dig example.com NS +short
```

### 4. See the route your data takes: `traceroute`

```bash
# Shows every router ("hop") between you and the destination
traceroute google.com
# Each line = one hop. Three times (ms) = three test probes.
# Lines with "* * *" mean that router didn't reply (often just a firewall policy).
```

### 5. Make your own HTTP request (act as the client)

```bash
# Fetch a web page and print the body
curl https://example.com

# Show ONLY the response headers (status code, server type, etc.)
curl -I https://example.com
# -I -> send a HEAD request (headers only)

# Verbose mode: watch DNS, TCP, TLS, and HTTP happen step by step
curl -v https://example.com
# Lines starting with '*' = connection details
# Lines starting with '>' = what YOU sent
# Lines starting with '<' = what the SERVER replied

# Send data to an API (a POST request with JSON)
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name": "devops-learner"}'
# -X POST -> use the POST method
# -H      -> add a header telling the server we're sending JSON
# -d      -> the body (data) of the request
```

### 6. Build a tiny server yourself in one line

```bash
# Go to any folder, then start a web server on port 8000
python3 -m http.server 8000
# This turns your laptop into a (very basic) web server!
# Open http://localhost:8000 in a browser to see the folder's files.
# Stop with Ctrl + C
```

Now **in a second terminal**, act as the client:

```bash
curl -v http://localhost:8000
# You'll see an HTTP request and response, live, between two programs
# on the same machine.
```

### 7. See which ports your machine is listening on

```bash
# Show all programs LISTENING for connections
ss -tulpn
# -t -> TCP     -u -> UDP
# -l -> only listening sockets
# -p -> show the program/process using the port
# -n -> show numbers (not names) for speed
# (you may need 'sudo' to see process names)

# Check specifically whether port 8000 is in use
ss -tulpn | grep 8000
```

### 8. Test whether a remote port is open

```bash
# Using netcat (nc): can I reach google.com on port 443?
nc -zv google.com 443
# -z -> just scan, don't send data
# -v -> verbose output
# "succeeded!" = port is open and reachable
# "Connection refused" or timeout = port is closed or blocked by a firewall
```

### 9. Connect to a remote server with SSH

```bash
# Basic form: ssh <username>@<server-ip-or-hostname>
ssh ubuntu@203.0.113.25

# Using a private key file (standard on cloud servers like AWS)
ssh -i ~/.ssh/my-key.pem ubuntu@203.0.113.25
# -i -> "identity file" (your private key)

# Key files MUST be private or SSH will refuse to use them
chmod 400 ~/.ssh/my-key.pem
# 400 = only YOU can read it; no one can write or execute it
```

> **SSH Analogy:** Your private key is like a physical house key. The server holds the matching lock (public key). Without the key, you're not getting in, and you must never hand your key to anyone.

### 10. Basic server information (once logged in)

```bash
hostname           # The server's name
uname -a           # Kernel and OS information
uptime             # How long it has been running + load average
df -h              # Disk space usage (-h = human readable: GB, MB)
free -h            # Memory (RAM) usage
nproc              # Number of CPU cores
```

### 11. Install and run a real web server (Nginx) on Ubuntu

```bash
# Refresh the list of available software
sudo apt update
# sudo = "run as administrator"

# Install the Nginx web server
sudo apt install -y nginx
# -y = automatically answer "yes" to prompts

# Start Nginx now, and make it start automatically on reboot
sudo systemctl start nginx
sudo systemctl enable nginx

# Check that it is running
sudo systemctl status nginx

# Test it locally (you should see the "Welcome to nginx!" HTML)
curl http://localhost
```

Then open the server's **public IP** in your browser. If it doesn't load, the cloud **firewall (Security Group)** is probably blocking port 80. This is the #1 beginner mistake!

### 12. Example: A simple firewall setup (UFW on Ubuntu)

```bash
# Default policy: block all incoming traffic, allow all outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH FIRST (otherwise you lock yourself out!)
sudo ufw allow 22/tcp

# Allow web traffic
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS

# Turn the firewall on
sudo ufw enable

# Review active rules
sudo ufw status verbose
```

---

## Real-World Use Case

### Scenario: Deploying a Node.js/Python API for a startup

You've built a REST API. Here's how everything in this README comes together in production.

```
                           ┌───────────────────────────────┐
   User's Browser          │         Cloud Provider         │
 (api.myapp.com) ───────►  │                                │
        │                  │  ┌──────────────────────────┐  │
        │ 1. DNS lookup    │  │   Web Server (Nginx)     │  │
        ▼                  │  │   Public IP: 54.x.x.x    │  │
  DNS: A record            │  │   Ports open: 80, 443    │  │
  api.myapp.com            │  └───────────┬──────────────┘  │
  → 54.x.x.x               │              │ forwards request│
                           │              ▼                 │
                           │  ┌──────────────────────────┐  │
                           │  │ App Server (your code)   │  │
                           │  │ Private IP: 10.0.1.5     │  │
                           │  │ Port 3000 (private only) │  │
                           │  └───────────┬──────────────┘  │
                           │              │                 │
                           │              ▼                 │
                           │  ┌──────────────────────────┐  │
                           │  │ Database (PostgreSQL)    │  │
                           │  │ Private IP: 10.0.2.8     │  │
                           │  │ Port 5432 (private only) │  │
                           │  └──────────────────────────┘  │
                           └───────────────────────────────┘
```

**Step-by-step, what the DevOps engineer does:**

1. **Provision a server:** Launch a Linux VM (e.g., AWS EC2) with a public IP.
2. **Register a domain & create DNS records:** Point `api.myapp.com` → the server's public IP using an **A record**.
3. **Secure access:** Log in only via **SSH keys** (no passwords). Restrict port 22 to the company's office IP.
4. **Install a web server (Nginx):** It receives the public traffic on ports 80/443, handles HTTPS encryption, and forwards requests to your app. This is called a **reverse proxy**.
5. **Run the app on a private port:** The app listens on port 3000 and is *not* exposed to the Internet directly.
6. **Isolate the database:** The database lives on a **private IP** and accepts connections *only* from the app server. Hackers on the Internet cannot even see it.
7. **Monitor:** Use `uptime`, `df -h`, `free -h`, and logs to ensure the server is healthy. Later, you'll automate this.

> **Analogy:** The web server is the hotel **reception** (public, anyone can walk up). The app server is the **staff-only kitchen**. The database is the **locked safe in the manager's office**. Guests talk only to reception. Reception talks to the kitchen. Only the kitchen can open the safe.

**Why this matters:** If you accidentally open port 5432 to `0.0.0.0/0` (the whole Internet), attackers will find and attack your database within hours. Understanding IPs, ports, and firewalls is what prevents this.

---

## Interview Questions & Answers

### Q1. What happens when you type `https://www.example.com` in your browser and press Enter?

**Answer:**
1. The browser checks its cache, then asks a **DNS resolver** to translate `www.example.com` into an IP address.
2. The browser opens a **TCP connection** to that IP on port **443** (3-way handshake).
3. A **TLS handshake** takes place to set up an encrypted connection and verify the server's certificate.
4. The browser sends an **HTTP request** (e.g., `GET /`).
5. The server (often a web server like Nginx, which may forward the request to an app and database) processes it and returns an **HTTP response** with a status code (e.g., `200 OK`) and the page content.
6. The browser renders the content.

---

### Q2. What is the difference between a public IP and a private IP?

**Answer:**
A **public IP** is globally unique and reachable from the Internet, so it's used for things that must be accessible to users, like a web server. A **private IP** (e.g., `10.x.x.x`, `172.16–31.x.x`, `192.168.x.x`) is only valid inside a private network and cannot be reached directly from the Internet. It is used for internal resources like databases and app servers. In production, we keep sensitive systems on private IPs to reduce the attack surface.

*Analogy: street address vs. an office desk extension.*

---

### Q3. What is DNS and what does an "A record" do?

**Answer:**
DNS (Domain Name System) translates human-friendly domain names into IP addresses. An **A record** maps a domain name to an **IPv4 address** (for example, `api.myapp.com → 54.210.12.7`). Other common records include **CNAME** (alias to another name), **MX** (mail servers), and **NS** (authoritative name servers).

---

### Q4. What is the difference between TCP and UDP? When would you use each?

**Answer:**
**TCP** is connection-oriented and reliable. It guarantees data arrives complete and in the right order, and re-sends anything lost. It's used for web traffic (HTTP/HTTPS), SSH, and databases. **UDP** is connectionless and faster but has no delivery guarantees. It's used where speed matters more than perfection, such as video calls, live streaming, online gaming, and DNS lookups.

*Analogy: TCP is a signed-for courier; UDP is dropping flyers in mailboxes.*

---

### Q5. A user says "the website is down." How would you start troubleshooting using what you know about servers and networking?

**Answer:**
I'd work from the outside in:
1. **DNS:** `dig example.com +short`. Does the domain resolve to the correct IP?
2. **Reachability:** `ping <ip>`, and `traceroute <ip>` to see if it's a network issue. (Note: some servers block ping, so a failed ping alone isn't conclusive.)
3. **Port check:** `nc -zv <ip> 443`. Is the web port open, or is a firewall / security group blocking it?
4. **HTTP response:** `curl -I https://example.com`. What status code do I get (`502`, `503`, `500`...)?
5. **Server health (via SSH):** Is the web server running (`systemctl status nginx`)? Is the disk full (`df -h`)? Is memory exhausted (`free -h`)? What do the logs say?

This layered approach quickly narrows down whether the problem is DNS, network, firewall, the web server, the application, or the machine's resources.

---

## Quick Recap Cheat Sheet

| Concept | One-Line Summary |
|---------|------------------|
| Internet | A global network of connected computers using shared rules (protocols) |
| Server | A computer (or program) that waits for requests and responds |
| Client | The one that sends the request |
| IP Address | The "street address" of a machine |
| Port | The "room number" of a program on that machine |
| DNS | The Internet's phone book: name → IP |
| TCP / UDP | Reliable-but-slower vs. fast-but-no-guarantee delivery |
| HTTP / HTTPS | Language of the web, without / with encryption |
| SSH | Secure remote login to a server |
| Firewall | Gatekeeper that decides which traffic is allowed |

---

**Previous:** Start of Course | **Next:** [Linux Introduction & Setup](../02-linux-introduction-and-setup/README.md)
