# Linux Introduction & Setup

> **Who is this for?** A backend developer who has used Windows or macOS for daily work but has never managed a Linux machine. By the end of this guide you will understand what Linux is, why almost every server runs it, and you will have a working Linux server you can log into.

---

## What is it and Why do we need it?

### What is Linux?

**Linux** is a free, open-source **operating system (OS)**. An operating system is the core software that sits between the physical hardware (CPU, memory, disk, network card) and the programs you run (your web server, your database, your code).

Strictly speaking, "Linux" is the name of the **kernel**, which is the very heart of the operating system. When people say "Linux", they usually mean the kernel plus a collection of tools and programs built around it.

> **Analogy: A Car**
> The **kernel** is the engine. It does the real work of turning fuel into motion, but you can't drive an engine by itself. A complete **Linux distribution** is the whole car: engine, steering wheel, dashboard, seats, and wipers. Different car makers (Ubuntu, Red Hat, Debian) build different cars around the same kind of engine.

### Why do we need Linux?

Here is the practical reality of the industry:

- The vast majority of web servers, cloud virtual machines, and container workloads run on Linux.
- It is **free**, so there are no licensing costs when you launch hundreds of servers.
- It is **stable**: servers routinely run for months without needing a reboot.
- It is **lightweight and scriptable**: everything can be done from a text terminal, which means everything can be automated. Automation is the heart of DevOps.
- It is **secure and transparent**: the source code is public, and a huge community reviews and patches it.
- Nearly every DevOps tool (Docker, Kubernetes, Ansible, Terraform, Jenkins) is built for Linux first.

> **Analogy: Why Restaurants Use Commercial Kitchens**
> A home kitchen (a desktop OS like Windows) is great for personal use: friendly, visual, convenient. A commercial kitchen (Linux) has no frills, is built for heavy continuous workloads, and every station can be configured exactly how the chef wants. You wouldn't run a 500-seat restaurant from a home kitchen, and you wouldn't usually run production infrastructure on a desktop OS.

### What does "Setup" mean here?

To learn Linux you need a Linux machine to practice on. This guide shows you three ways to get one:

1. A **cloud server** (AWS EC2), which is closest to real DevOps work.
2. A **local virtual machine** on your own computer.
3. **WSL** (Windows Subsystem for Linux), if you are on Windows.

---

## Core Concepts

### 1. Kernel vs. Operating System vs. Distribution

| Term | What it is | Analogy |
|------|-----------|---------|
| **Kernel** | The core that manages hardware, memory, processes, and devices | The car's engine |
| **Operating System** | Kernel + system tools + libraries | The engine plus the chassis and wiring |
| **Distribution (distro)** | A packaged, ready-to-use Linux OS made by a vendor or community | A finished car model sold by a manufacturer |

The kernel itself was created by **Linus Torvalds** in 1991. The surrounding tools largely came from the **GNU project**, which is why you will sometimes see the name "GNU/Linux".

---

### 2. What the Kernel Actually Does

The kernel has four main jobs:

| Job | Meaning | Analogy |
|-----|---------|---------|
| **Process management** | Decides which program uses the CPU and for how long | A traffic controller deciding which car goes through the junction |
| **Memory management** | Hands out RAM to programs and keeps them from interfering with each other | A hotel assigning rooms so guests don't walk into each other's rooms |
| **Device management** | Talks to hardware (disks, network cards, keyboards) through drivers | A translator between you and a foreign-speaking contractor |
| **File system management** | Organizes how data is stored and retrieved on disks | A librarian who knows exactly which shelf every book is on |

---

### 3. Linux Distributions (Distros)

Because Linux is open source, anyone can package it. These packages are called **distributions**. They differ mainly in **who maintains them**, **how often they update**, and **which package manager they use**.

| Family | Popular Distros | Package Manager | Commonly used for |
|--------|-----------------|-----------------|-------------------|
| **Debian family** | Ubuntu, Debian | `apt` / `dpkg` | General servers, cloud, development. Ubuntu is the most beginner-friendly choice |
| **Red Hat family** | RHEL, Rocky Linux, AlmaLinux, Fedora, Amazon Linux | `dnf` / `yum` / `rpm` | Enterprise environments, AWS (Amazon Linux) |
| **Alpine** | Alpine Linux | `apk` | Very small container images |
| **Arch / SUSE and others** | Arch, openSUSE | `pacman` / `zypper` | Specialist use |

> **Note:** CentOS Linux (the old free clone of RHEL) has been discontinued. **Rocky Linux** and **AlmaLinux** are the popular replacements for those who want a RHEL-compatible system.

**Which one should you pick?** For this course, use **Ubuntu Server LTS**. LTS stands for **Long Term Support**: a version that receives security updates for many years, which is exactly what you want on a server.

> **Analogy:** Distros are like different flavors of the same ice cream base. The base (kernel) is identical. The toppings and packaging (tools, defaults, update schedule) differ.

---

### 4. The Terminal, Shell, and Command Line

| Term | Meaning | Analogy |
|------|---------|---------|
| **Terminal** | The window where you type commands | The phone handset |
| **Shell** | The program inside the terminal that reads and runs your commands. The most common is **Bash** | The operator who understands what you say and carries it out |
| **Command** | A single instruction, such as `ls` or `cd` | One sentence you speak to the operator |

Servers usually have **no graphical interface (GUI)**. You control them entirely through the shell, typically over **SSH** (a secure remote connection, introduced in Topic 1).

---

### 5. The Linux Directory Structure

Linux has **one single tree of folders** that starts at the **root**, written as `/`. There are no drive letters like `C:\`. Everything, including disks and devices, hangs off this one tree.

```
/
├── bin      Essential command programs (ls, cp, cat)
├── boot     Files needed to start the system (the kernel)
├── dev      Device files (disks, terminals)
├── etc      Configuration files for the system and installed software
├── home     Personal folders for regular users (/home/alice)
├── lib      Shared libraries needed by programs
├── opt      Optional, third-party software
├── proc     Live information about running processes (virtual)
├── root     The home folder of the administrator (root) user
├── tmp      Temporary files (often cleared on reboot)
├── usr      User programs, utilities, and documentation
└── var      Variable data that changes: logs, caches, databases
```

> **Analogy: A Well-Organized Office Building**
> `/etc` is the filing cabinet of rules and settings. `/home` is where each employee has their own desk. `/var/log` is the security desk's visitor logbook. `/tmp` is the recycling bin by the door. `/root` is the manager's private office.

**The directories you will visit most as a DevOps engineer:** `/etc` (configs), `/var/log` (logs), `/home` (user files), and `/tmp` (scratch space).

---

### 6. Everything Is a File

A core Linux philosophy: nearly everything is represented as a file, including documents, folders, hardware devices, and even running processes. This means the same simple tools (`cat`, `ls`, `grep`) work in many different situations.

> **Analogy:** Think of a universal remote. Whether you are controlling the TV, the speakers, or the lights, you use the same kind of button presses.

---

### 7. Users and the Root Account

- A **regular user** has limited permissions and can only change its own files.
- The **root** user is the all-powerful administrator who can change anything, including deleting the entire system.
- Best practice: **log in as a regular user** and use **`sudo`** ("superuser do") to run individual commands with administrator rights only when needed.

> **Analogy:** Employees carry normal key cards. The master key (root) is kept in a safe. When you need it for a specific task (`sudo`), you sign it out temporarily, use it, and give it back.

---

### 8. Package Managers

A **package manager** installs, updates, and removes software for you, including all the other software it depends on. It downloads from trusted online collections called **repositories**.

> **Analogy:** It is the **app store** for your server. Instead of hunting for installers on random websites, you ask the store to install an app, and it handles the rest.

| Task | Ubuntu / Debian (`apt`) | RHEL / Amazon Linux / Rocky (`dnf`) |
|------|------------------------|--------------------------------------|
| Refresh the list of available software | `sudo apt update` | `sudo dnf check-update` |
| Install a package | `sudo apt install nginx` | `sudo dnf install nginx` |
| Upgrade installed packages | `sudo apt upgrade` | `sudo dnf upgrade` |
| Remove a package | `sudo apt remove nginx` | `sudo dnf remove nginx` |
| Search for a package | `apt search nginx` | `dnf search nginx` |

---

### 9. Services and systemd

A **service** (or **daemon**) is a program that runs quietly in the background, like a web server or a database. Most modern Linux distributions use **systemd** to start, stop, and supervise these services, using the `systemctl` command.

> **Analogy:** systemd is the **building manager** who turns the lights on in the morning, makes sure the elevators keep running, and restarts anything that breaks.

---

### 10. Ways to Set Up a Linux Machine

| Option | Best for | Cost | Closeness to real DevOps work |
|--------|----------|------|-------------------------------|
| **AWS EC2 (cloud VM)** | Real-world practice, remote access, networking labs | Free tier available (limits apply) | Highest |
| **Local VM (Multipass / VirtualBox)** | Offline practice, safe experiments | Free | Medium |
| **WSL on Windows** | Quick command-line practice | Free | Lower (no real server networking or boot) |

---

## Practical Examples & Commands

### Option A: Launch a Linux Server on AWS EC2 (Recommended)

#### Step 1: Launch the instance (AWS Console)

1. Sign in to the AWS Console and open **EC2**.
2. Click **Launch instance**.
3. **Name:** `devops-lab-01`
4. **Application and OS Image (AMI):** choose **Ubuntu Server 24.04 LTS** (or the latest Ubuntu LTS shown).
5. **Instance type:** choose a small, free-tier-eligible type such as `t3.micro` (or `t2.micro` in regions where that is the free-tier option). Always check the current free-tier terms on the AWS website, because they change.
6. **Key pair:** click **Create new key pair**, name it `devops-key`, type **RSA**, format **.pem**. The file downloads once. **Keep it safe; you cannot download it again.**
7. **Network settings:** create a security group that allows **SSH (port 22) from "My IP"** only. Avoid `0.0.0.0/0` for SSH, since that exposes login attempts to the entire Internet.
8. **Storage:** the default 8 GB is fine.
9. Click **Launch instance**, then wait until **Instance state** says `Running`.
10. Copy the **Public IPv4 address** from the instance details.

#### Step 2: Connect with SSH

```bash
# Move the key to a safe place (adjust the path to wherever it downloaded)
mkdir -p ~/.ssh
mv ~/Downloads/devops-key.pem ~/.ssh/

# SSH refuses keys that other people can read, so lock permissions down
chmod 400 ~/.ssh/devops-key.pem
# 400 = owner can read; nobody can write or execute; group/others get nothing

# Connect. Replace 203.0.113.25 with YOUR instance's public IP
ssh -i ~/.ssh/devops-key.pem ubuntu@203.0.113.25
# -i               -> use this private key file to prove who you are
# ubuntu           -> default username on Ubuntu AMIs
# (Amazon Linux uses 'ec2-user', Debian uses 'admin', RHEL uses 'ec2-user')
# 203.0.113.25     -> the server's public IP address
```

The first time, SSH asks whether you trust the server's fingerprint. Type `yes`. If you see a prompt such as `ubuntu@ip-172-31-x-x:~$`, you are inside your Linux server.

#### Step 3: Stop the instance when you are done

Instances may cost money while running. From the console, choose **Instance state > Stop** (or **Terminate** to delete it entirely) when you are finished practicing. Also consider setting up an AWS billing alert.

---

### Option B: A Local VM with Multipass

Multipass (from Canonical, the makers of Ubuntu) is the quickest way to get a fresh Ubuntu server on your own machine.

```bash
# Create and start a new Ubuntu VM named "devops-lab"
multipass launch --name devops-lab --cpus 2 --memory 2G --disk 10G
# --cpus 2     -> give the VM 2 CPU cores
# --memory 2G  -> give it 2 GB of RAM
# --disk 10G   -> give it a 10 GB disk

# List your VMs and their IP addresses
multipass list

# Open a shell inside the VM
multipass shell devops-lab

# From your own machine: stop, start, and delete the VM
multipass stop devops-lab
multipass start devops-lab
multipass delete devops-lab && multipass purge
```

---

### Option C: WSL on Windows

```powershell
# Run this in PowerShell as Administrator
wsl --install -d Ubuntu
# Installs the Windows Subsystem for Linux and the Ubuntu distribution.
# Restart if asked, then open "Ubuntu" from the Start menu and create a username and password.
```

---

### First Commands After You Log In

```bash
# 1. Who am I, and where am I?
whoami                 # Prints your username (e.g. ubuntu)
hostname               # Prints the machine's name
pwd                    # "Print Working Directory": shows your current folder

# 2. What OS and kernel is this?
cat /etc/os-release    # Shows the distribution name and version
uname -r               # Shows the kernel version
uname -a               # Shows all system information in one line

# 3. How is the machine doing?
uptime                 # How long it has been running + load average
df -h                  # Disk space (-h = human-readable, e.g. GB)
free -h                # Memory (RAM) usage
nproc                  # Number of CPU cores

# 4. Look around the filesystem
ls /                   # List top-level directories
ls -l /etc | head     # List the first lines of /etc in long format
cd /var/log && ls     # Move into the logs folder and list it
cd ~                   # Return to your home folder (~ is shortcut for /home/<you>)
```

---

### Keep the System Updated (Always Do This First)

```bash
# Download the newest list of available packages and versions
sudo apt update

# Install all available updates for packages already on the system
sudo apt upgrade -y
# -y = automatically answer "yes" to confirmation prompts

# Some updates (especially the kernel) need a reboot to take effect
sudo reboot
# Your SSH session will disconnect. Wait about a minute, then reconnect.
```

---

### Install and Verify Some Common Tools

```bash
# Install useful basics in one command
sudo apt install -y curl wget git vim htop unzip
# curl/wget -> download things from the web
# git       -> version control
# vim       -> text editor available in the terminal
# htop      -> a friendly live view of running processes
# unzip     -> extract .zip files

# Confirm they installed
git --version
curl --version | head -n 1
```

---

### Get Help Inside Linux

```bash
man ls                 # Open the full manual page for 'ls' (press q to quit)
ls --help              # Quick summary of options
which curl             # Show where a command's program file lives
type cd                # Show whether something is a program or built into the shell
history | tail         # Show your last typed commands
```

---

### Create a Regular Admin User (Good Practice)

```bash
# Create a new user called "devops" (you will be asked to set a password)
sudo adduser devops

# Give the user administrator rights via the 'sudo' group
sudo usermod -aG sudo devops
# usermod -> modify a user account
# -aG     -> APPEND to the supplementary Group (without -a, existing groups would be replaced!)

# Verify the membership
groups devops
```

(Users and groups are covered in depth in Topic 5.)

---

## Real-World Use Case

### Scenario: Preparing a brand-new server for a team's application

A new project needs a Linux server to host its backend API. Here is what a DevOps engineer typically does in the first 30 minutes.

```
   Engineer's Laptop
         |
         |  SSH (port 22, key-based login)
         v
 +---------------------------------------------+
 |  Cloud VM: Ubuntu Server 24.04 LTS          |
 |                                             |
 |  1. Update the OS      (apt update/upgrade) |
 |  2. Create admin user  (adduser + sudo)     |
 |  3. Lock down SSH      (keys only, no root) |
 |  4. Enable a firewall  (allow 22, 80, 443)  |
 |  5. Install software   (nginx, runtime)     |
 |  6. Set up monitoring and log checks        |
 +---------------------------------------------+
```

**Step-by-step:**

1. **Launch** the VM from a trusted, current **LTS image** in the cloud provider.
2. **Patch it** immediately:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
3. **Create a named admin account** for each engineer rather than sharing one login. This provides accountability, meaning you can tell who did what.
4. **Harden SSH** by editing the SSH configuration:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   Set these values (each line explained):
   ```text
   PermitRootLogin no            # Nobody can log in directly as root
   PasswordAuthentication no     # Only SSH keys are accepted, not guessable passwords
   ```
   Then apply the change and keep your existing session open while you test a second one:
   ```bash
   sudo systemctl restart ssh
   ```
5. **Enable the firewall** so only the needed ports are open:
   ```bash
   sudo ufw allow 22/tcp     # Allow SSH FIRST so you don't lock yourself out
   sudo ufw allow 80/tcp     # Allow HTTP
   sudo ufw allow 443/tcp    # Allow HTTPS
   sudo ufw enable           # Turn the firewall on
   ```
6. **Install the application stack** (web server, language runtime, and so on), enable it to start on boot, and verify it with `systemctl status` and `curl`.
7. **Document and automate.** In mature teams, these steps are captured in scripts or tools (shell scripts, Ansible, Terraform, cloud-init) so every new server is built identically.

**Why this matters:** Servers exposed to the Internet are probed by automated bots within minutes of being created. A freshly launched server with password logins enabled and no updates is an easy target. Knowing basic Linux setup and hardening is the first line of defense.

> **Analogy:** Moving into a new office. Before the team arrives, you change the locks, install the alarm, set up the fire exits, and only then bring in the desks and computers.

---

## Interview Questions & Answers

### Q1. What is Linux, and what is the difference between the Linux kernel and a Linux distribution?

**Answer:**
Linux is a free, open-source operating system family. Strictly, **Linux is the kernel**, the core component that manages hardware, memory, processes, and file systems. A **distribution** (such as Ubuntu, Debian, or Red Hat Enterprise Linux) bundles the Linux kernel with system tools, libraries, a package manager, and default configuration to make a complete, usable OS.

*Analogy: the kernel is the engine; the distribution is the whole car.*

---

### Q2. Why is Linux so widely used for servers?

**Answer:**
Linux is free (no licence fees), stable (can run for long periods without a reboot), secure and open to review, lightweight, and highly customizable. It can be fully managed through the command line, which makes **automation** easy, and automation is central to DevOps. In addition, most cloud platforms and DevOps tools (Docker, Kubernetes, Ansible, Terraform) are designed with Linux in mind.

---

### Q3. What does "LTS" mean in Ubuntu, and why does it matter for servers?

**Answer:**
LTS means **Long Term Support**. An LTS release receives security patches and maintenance updates for several years (typically five years, with extended options). It matters for servers because production systems value **stability and predictability** over new features. You don't want to rebuild your servers every six months.

---

### Q4. Explain the Linux directory structure. Where would you look for configuration files and logs?

**Answer:**
Linux has a single directory tree that starts at the root `/`. Key directories include `/etc` (configuration files), `/var/log` (log files), `/home` (users' personal files), `/bin` and `/usr/bin` (commands and programs), `/tmp` (temporary files), `/boot` (boot files and kernel), and `/root` (the administrator's home). For configuration, I'd check `/etc`; for troubleshooting, I'd check `/var/log`.

---

### Q5. Why should you avoid working as the root user, and what do you use instead?

**Answer:**
The root user has unrestricted power, so a simple typing mistake (like deleting the wrong folder) can destroy the entire system, and a compromised root session gives an attacker full control. Best practice is to log in as a **regular user** and use **`sudo`** to run only specific commands with elevated privileges. `sudo` also **logs** who ran what, which supports auditing and accountability. On cloud servers, we usually disable direct root login over SSH completely.

---

## Quick Recap Cheat Sheet

| Concept | One-Line Summary |
|---------|------------------|
| Linux | A free, open-source OS kernel, the basis of most servers |
| Distribution | A packaged Linux OS (Ubuntu, Debian, RHEL, Amazon Linux) |
| LTS | Long Term Support release, preferred for servers |
| Shell / Terminal | The command-line interface you type into |
| `/etc`, `/var/log`, `/home` | Configs, logs, user files |
| Package manager | The app store (`apt`, `dnf`) |
| `sudo` | Run a single command with admin rights |
| systemd / `systemctl` | Manages background services |
| SSH | Secure remote login to your server |

---

**Previous:** [Internet and Servers Basics](../01-internet-and-servers-basics/README.md) | **Next:** [Linux Basic Commands for DevOps Engineers](../03-linux-basic-commands/README.md)
