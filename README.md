# DevOps Learning Journey: Linux Fundamentals
<img width="610" height="327" alt="images__8_-removebg-preview" src="https://github.com/user-attachments/assets/cce963b8-edc7-44ac-a7ed-1d219c571002" />




A beginner-friendly, hands-on documentation of my DevOps learning journey, starting with Linux. Each topic lives in its own folder with a standalone `README.md` that explains the concept in plain English, uses real-world analogies, shows commented commands, and ends with interview preparation.

---

## Who is this for?

- Backend or application developers who have never worked with servers or infrastructure.
- Anyone starting a career in DevOps, Cloud, or Site Reliability Engineering (SRE).
- Learners preparing for junior DevOps or Linux administration interviews.

No prior operations experience is assumed. Jargon is either avoided or defined the first time it appears.

---

## How each topic is structured

Every topic README follows the same five-section layout, so you always know where to look:

| Section | What you will find |
|---------|--------------------|
| What is it and Why do we need it? | A plain-language explanation of the topic, plus a real-world analogy |
| Core Concepts | The theory broken down into small, digestible pieces |
| Practical Examples & Commands | Exact commands and configuration snippets, with comments on every line |
| Real-World Use Case | How the topic is used in a production environment |
| Interview Questions & Answers | 3 to 5 common interview questions with concise, confident answers |

---

## Table of Contents

| # | Topic | Key Skills Covered | Status |
|---|-------|--------------------|--------|
| 01 | [Internet and Servers Basics](./01-internet-and-servers-basics/README.md) | How the Internet works, IPs, DNS, ports, protocols, what servers are | Complete |
| 02 | [Linux Introduction & Setup](./02-linux-introduction-and-setup/README.md) | What Linux is, distributions, setting up a Linux server | Planned |
| 03 | [Linux Basic Commands for DevOps Engineers](./03-linux-basic-commands/README.md) | Navigation, files and directories, viewing and editing content | Planned |
| 04 | [Advanced Linux Commands for DevOps](./04-advanced-linux-commands/README.md) | Processes, services, logs, system monitoring, package management | Planned |
| 05 | [Users and Group Management](./05-users-and-group-management/README.md) | Creating users and groups, sudo, password policies | Planned |
| 06 | [File Management in Linux](./06-file-management/README.md) | Permissions, ownership, umask, zip, tar | Planned |
| 07 | [File Transfer Commands](./07-file-transfer-commands/README.md) | scp, rsync | Planned |
| 08 | [Linux Networking Commands](./08-linux-networking-commands/README.md) | ping, traceroute, telnet, netstat, curl, wget, ARP, Nmap, route tables | Planned |
| 09 | [Pro Linux Commands: AWK, GREP, FIND, SED](./09-pro-linux-commands-awk-grep-find-sed/README.md) | Searching, filtering, and transforming text and files | Planned |
| 10 | [Linux Volume Management](./10-linux-volume-management/README.md) | AWS EBS volumes, mounting, filesystem creation | Planned |
| 11 | [LVM (Logical Volume Manager) in Linux](./11-lvm-logical-volume-manager/README.md) | Physical Volumes, Volume Groups, Logical Volumes, extending storage | Planned |

---

## Suggested Learning Path

The topics build on each other, so follow them in order:

1. **Foundations (Topic 01 to 02):** Understand how servers and the Internet work, then meet Linux and set up your own server.
2. **Daily Driver Skills (Topic 03 to 04):** Learn the commands you will type every single day.
3. **Access and Security (Topic 05 to 06):** Control who can do what on a system.
4. **Moving and Connecting (Topic 07 to 08):** Transfer files and troubleshoot networks.
5. **Power Tools (Topic 09):** Process logs and text like a professional.
6. **Storage (Topic 10 to 11):** Manage disks, volumes, and flexible storage on production servers.

---

## Repository Structure

```
.
├── README.md                                    # You are here
├── 01-internet-and-servers-basics/
│   └── README.md
├── 02-linux-introduction-and-setup/
│   └── README.md
├── 03-linux-basic-commands/
│   └── README.md
├── 04-advanced-linux-commands/
│   └── README.md
├── 05-users-and-group-management/
│   └── README.md
├── 06-file-management/
│   └── README.md
├── 07-file-transfer-commands/
│   └── README.md
├── 08-linux-networking-commands/
│   └── README.md
├── 09-pro-linux-commands-awk-grep-find-sed/
│   └── README.md
├── 10-linux-volume-management/
│   └── README.md
└── 11-lvm-logical-volume-manager/
    └── README.md
```

---

## Prerequisites

To follow along with the practical examples, you will need:

- A computer with a terminal (Linux, macOS, or Windows with WSL).
- A Linux environment to practice in. Any one of the following works:
  - A cloud VM (for example, an AWS EC2 free-tier instance running Ubuntu)
  - A local virtual machine (VirtualBox, VMware, or Multipass)
  - Windows Subsystem for Linux (WSL)
- A basic text editor and a willingness to break things in a safe environment.

> Tip: Always practice on a throwaway machine, not a production server. Breaking things is one of the fastest ways to learn.

---

## How to Use This Repository

1. Clone the repository:
   ```bash
   git clone https://github.com/<your-username>/<your-repo-name>.git
   cd <your-repo-name>
   ```
2. Open the topic folder you want to study and read its `README.md`.
3. Run every command yourself. Do not just read them.
4. Test yourself with the interview questions at the end of each topic.
5. Revisit topics as needed. Repetition builds muscle memory.

---

## Conventions Used

- Commands are shown in fenced code blocks, with comments explaining what each part does.
- Lines beginning with `#` inside a code block are comments, not commands.
- `sudo` means "run this command with administrator privileges".
- Placeholders such as `<username>` or `<server-ip>` should be replaced with your own values.
r-username](https://github.com/your-username)

If this repository helped you, consider giving it a star.
