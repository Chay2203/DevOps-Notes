# DevOps Learning Guide — Week One

A structured reference for AWS infrastructure, Linux fundamentals, DevOps practices, shell scripting, and Git.

---

## Table of Contents

1. [Part 1: AWS Infrastructure](#part-1-aws-infrastructure)
2. [Part 2: Linux Essentials](#part-2-linux-essentials)
3. [Part 3: DevOps & Microservices](#part-3-devops--microservices)
4. [Part 4: Shell Scripting](#part-4-shell-scripting)
5. [Part 5: Version Control with Git](#part-5-version-control-with-git)

---

# Part 1: AWS Infrastructure

## Regions and Availability Zones

**AWS Regions** are separate geographic areas where AWS runs data centers. Each region is independent, which helps with:

- **Compliance** — Keep data in specific countries or regions
- **Resilience** — Isolate failures to one region
- **Latency** — Run workloads closer to users

Each region has its own set of services: EC2, RDS, S3, etc.

**Availability Zones (AZs)** are one or more data centers inside a region, connected by fast, low-latency links. A region usually has 2–6 AZs. Use multiple AZs for high availability.

---

## Virtual Private Cloud (VPC)

A **VPC** is your private network inside an AWS region—like your own virtual data center. You can have multiple VPCs per region.

Inside a VPC you control:

| Component | Purpose |
|-----------|---------|
| **CIDR blocks** | IP address range for the VPC |
| **Subnets** | Smaller segments (often per AZ) |
| **Route tables** | How traffic is routed |
| **Internet Gateway** | Outbound/inbound internet access |
| **Security Groups & NACLs** | Inbound/outbound traffic rules |

---

## Subnets

**Subnets** split a VPC into smaller networks. Each subnet lives in a single AZ.

**Design tips:**

1. Spread workloads across multiple AZs.
2. Use tiers: public (web), private (app), data (DB).
3. Use NAT Gateways so private subnets can reach the internet.
4. Plan CIDR ranges for growth.
5. Tag consistently (e.g. `Environment=Production`, `Tier=Web`).
6. Combine route tables and security groups for isolation.

---

## EC2 Launch Configuration

### 1. Name and tags

- **Name** — Sets the `Name` tag; highly recommended.
- **Custom tags** — e.g. `Environment=Production`, `Role=WebServer` for billing, automation, and organization.

### 2. AMI (Amazon Machine Image)

- Defines OS and pre-installed software (Amazon Linux, Ubuntu, Windows, etc.).
- **Free tier:** Prefer “Free tier eligible” AMIs.
- **Sources:** Quick Start, AWS Marketplace, Community AMIs.

### 3. Instance type

- Sets vCPUs, memory, network. Examples: `t3.micro` (light), `m5.large` (general purpose).
- Free tier often limits to `t2.micro` / `t3.micro`.

### 4. Key pair

- Used for SSH (Linux) or RDP (Windows). Required for secure access; skipping it can lock you out.

### 5. Network

- **VPC / Subnet** — Where the instance runs.
- **Auto-assign Public IP** — For direct internet reachability.
- **Security groups** — Inbound/outbound firewall rules.
- Optional: extra network interfaces.

### 6. Storage

- **Root volume** — From AMI; you can add EBS or instance store.
- **Types** — e.g. gp3, gp2, io1, io2, st1 (performance vs cost).
- **Size, IOPS, throughput** — Per volume type.
- **Delete on termination** — Avoid orphan volumes.
- **Encryption** — Optional, with KMS.
- **Device name** — e.g. `/dev/xvda`, `/dev/sdb`.

### 7. Advanced details

- **IAM instance profile** — Role for S3, other AWS APIs.
- **Shutdown behavior** — Stop vs terminate.
- **Termination protection** — Prevents accidental terminate.
- **Detailed CloudWatch** — 1‑minute metrics (extra cost).
- **Placement group** — Clustered / spread / partition.
- **Tenancy** — Shared vs dedicated.
- **User data** — Script/cloud-init at first boot.
- **Metadata (IMDS)** — Version and security options.
- **Tag specifications** — Tags for instance, volumes, ENIs.

---

# Part 2: Linux Essentials

## Why Linux for DevOps

Linux runs most servers, cloud VMs, containers, and CI/CD systems. The **kernel** manages CPU, memory, devices, and security; the **shell** (e.g. Bash, Zsh) is your command-line interface. Knowing the CLI enables automation, scripting, and troubleshooting.

---

## 1. Basic commands

| Command | Purpose |
|---------|---------|
| `pwd` | Print current directory |
| `cd <path>` | Change directory (`cd ..`, `cd ~`) |
| `mkdir [name]` | Create directory (`mkdir -p a/b/c` for nested) |
| `rmdir [name]` | Remove empty directory |
| `touch <file>` | Create empty file or update timestamp |
| `cat <file>` | Show or concatenate file contents |
| `echo "text"` | Print text; `echo "x" > f` writes to file |
| `clear` | Clear screen |
| `history` | List command history |

**Lab 1:** Create `starting_point`, `cd` into it, create `secret_lair.txt`, run `pwd`, then `history`.

---

## 2. File management

| Command | Purpose |
|---------|---------|
| `ls` | List files (`-l` long, `-a` hidden, `-lh` human-readable) |
| `cp <src> <dest>` | Copy (`-r` for directories) |
| `mv <src> <dest>` | Move or rename |
| `rm <file>` | Remove (`-r` for directories — no recycle bin) |
| `find <path> -name "*.log"` | Find by name; `-type f`, `-size +50M` |
| `du -sh <dir>` | Directory size |
| `df -h` | Disk free space |

**Lab 2:** Remove `delete_directory` and `delete.txt`, find `.txt` under `/home/user`, check `/var/log/` size with `du`.

---

## 3. Process management

| Command | Purpose |
|---------|---------|
| `ps` | Current user processes; `ps -ef` or `ps aux` for all |
| `top` | Live view (q quit, k kill, r renice) |
| `htop` | Friendlier top (install: `apt install htop -y`) |
| `kill <PID>` | Terminate (default SIGTERM); `kill -9 <PID>` force |
| `jobs`, `fg %1`, `bg` | Background/foreground jobs |

**Lab 3:** List processes, find `sshd` PID, run `sleep 60 &`, bring to foreground with `fg`, then kill by PID.

---

## 4. Permissions and ownership

```bash
chmod 755 file.sh      # rwxr-xr-x
chown user:group file  # ownership
chgrp group file       # group only
```

**Lab 4 (secret_lair):** Count subdirectories, find the permission-restricted one, change ownership, write answers to `/home/user/answer.txt`.

---

## 5. ACLs (Access Control Lists)

ACLs add extra users/groups to a file or directory beyond the single owner/group/others model.

| Task | Command |
|------|---------|
| Check support | `mount \| grep acl` |
| View ACLs | `getfacl <file>` |
| Set user ACL | `setfacl -m u:john:r-- file` |
| Set group ACL | `setfacl -m g:devs:rw- file` |
| Remove one entry | `setfacl -x u:john file` |
| Remove all ACLs | `setfacl -b file` |
| Recursive on dir | `setfacl -R -m u:john:rwX /var/www/` |
| Default ACL (new files) | `setfacl -d -m u:john:rwX /var/www/` |
| Mask (max effective) | `setfacl -m m::r-- file` |

Use `X` (capital) for execute-only-on-directories when using `-R`.

---

# Part 3: DevOps & Microservices

## What is DevOps?

**DevOps** connects development and operations so software can be delivered and operated in a fast, reliable way. It combines culture, practices, and tools.

- **Culture:** Shared ownership, fewer silos.
- **Practices:** Automation, CI/CD, monitoring, feedback.
- **Contrast:** Old model = handoff to Ops → slow fixes; DevOps = code triggers pipelines → automated test and deploy.

**Why adopt DevOps:**

1. **Faster delivery** — Releases in days, not months.
2. **Collaboration** — Dev and Ops both own infra and app.
3. **Reliability** — Automated tests and monitoring catch issues early.
4. **Automation** — Build, deploy, provision via Jenkins, GitHub Actions, etc.
5. **Scalability** — Cloud and Kubernetes scale on demand.
6. **Continuous improvement** — Data from monitoring drives optimization.

---

## DevOps Lifecycle

A continuous loop:

1. **Plan** — Requirements, backlog, priorities.
2. **Code** — Develop in small, frequent changes.
3. **Build** — Compile, package, create artifacts.
4. **Test** — Automated unit, integration, e2e tests.
5. **Release** — Versioning, release notes, approvals.
6. **Deploy** — Roll out to staging/production (often automated).
7. **Operate** — Run and maintain in production.
8. **Monitor** — Metrics, logs, alerts; feed back into Plan and Code.

---

## DevOps Principles

- **Collaboration** — Dev and Ops work together end-to-end.
- **Automation** — Eliminate manual, repetitive steps.
- **CI/CD** — Integrate and deliver continuously.
- **Measurement** — Use metrics to improve.
- **Sharing** — Knowledge, tools, and feedback across teams.
- **Infrastructure as Code (IaC)** — Version and automate infra.
- **Monitoring & feedback** — Observability and quick feedback loops.

---

## DevOps Practices

- **Continuous Integration** — Frequent commits, automated builds and tests.
- **Continuous Delivery/Deployment** — Automate release and deploy.
- **Automated testing** — Unit, integration, security, performance.
- **Infrastructure as Code** — Terraform, CloudFormation, Ansible.
- **Monitoring & logging** — Prometheus, Grafana, ELK, CloudWatch.
- **Configuration management** — Ansible, Chef, Puppet.

**Common tools:** Git, Jenkins, Docker, Kubernetes, Terraform, Prometheus, Grafana, Ansible.

---

## Microservices

A **microservices** architecture splits an application into small, loosely coupled services. Each service owns one business capability (auth, payments, orders) and talks via APIs. Unlike a monolith, services can be developed, deployed, and scaled independently.

---

## DNS basics

**DNS** maps names (e.g. `example.com`) to IPs.

1. Browser asks for `example.com`.
2. Resolver asks Root → TLD (.com) → Authoritative server.
3. Authoritative server returns IP.
4. Browser connects to that IP.

**Cloud DNS:** AWS Route 53, Azure DNS, Cloudflare — offer health checks, failover, geo routing.

---

## Load balancers

Distribute traffic across backends for availability and scale.

| Type | Layer | Protocol | Examples | Use |
|------|--------|----------|----------|-----|
| **L4** | Transport | TCP/UDP | NLB, HAProxy | Fast, network-level |
| **L7** | Application | HTTP/HTTPS | ALB, Nginx, Traefik | URL, headers, cookies |

Features: health checks, sticky sessions, SSL termination, auto-scaling.

---

## Sync vs async communication

| | Synchronous | Asynchronous |
|--|-------------|---------------|
| **Model** | Caller waits for response | Caller sends and continues |
| **Example** | REST API | Kafka, RabbitMQ |
| **Protocol** | HTTP/HTTPS | AMQP, MQTT, Kafka |
| **Use** | Real-time (e.g. login) | Decoupled (notifications, billing) |
| **Trade-off** | Coupling, latency | Complexity, eventual consistency |

```
Sync:  Client → Service A → [wait] → Response
Async: Client → Service A → Queue → Service B (processes later)
```

---

## SQL vs NoSQL

| | SQL | NoSQL |
|--|-----|--------|
| **Structure** | Tables, rows, columns | Key-value, document, graph, wide-column |
| **Schema** | Fixed | Flexible |
| **Scaling** | Vertical | Horizontal |
| **Examples** | MySQL, PostgreSQL | MongoDB, DynamoDB, Cassandra |
| **Use** | Transactions, structured data | Scale, flexible schema |

Microservices often use both (polyglot persistence).

---

## API Gateway

Sits between clients and backend services. Handles routing, auth, rate limiting, SSL. Examples: AWS API Gateway, Kong, Nginx, Istio, Apigee.

---

## Centralized logging

Containers/pods are ephemeral; local logs disappear. Centralize with:

| Role | Tools |
|------|--------|
| Collect | Fluentd, Fluent Bit, Logstash |
| Store/index | Elasticsearch, Loki |
| Visualize | Kibana, Grafana |
| Cloud | CloudWatch, Stackdriver, Dynatrace |

Enables search, debugging, and log-based alerting.

---

## 12-Factor App

Principles for portable, scalable, cloud-native apps (good for microservices and containers):

1. **Codebase** — One repo per app; many deploys.
2. **Dependencies** — Declare explicitly; never rely on system packages.
3. **Config** — Store config in environment, not code.
4. **Backing services** — Treat DB, queue, etc. as attached resources.
5. **Build, release, run** — Strict separation of build, release, and run stages.
6. **Processes** — Stateless processes; state in backing services.
7. **Port binding** — Export services via port binding.
8. **Concurrency** — Scale out via process model.
9. **Disposability** — Fast startup, graceful shutdown.
10. **Dev/prod parity** — Keep environments as similar as possible.
11. **Logs** — Treat logs as event streams.
12. **Admin processes** — Run admin tasks as one-off processes.

---

# Part 4: Shell Scripting

## Hello World and shebang

```bash
#!/bin/bash
echo "Hello, World!"
```

**Shebang** (`#!`) tells the OS which interpreter to use.

| Shebang | Interpreter | Use |
|---------|-------------|-----|
| `#!/bin/bash` | Bash | General scripting |
| `#!/bin/sh` | sh (often dash) | Portable, POSIX |
| `#!/usr/bin/env bash` | Bash from PATH | Portable path |
| `#!/usr/bin/python3` | Python | Python scripts |

---

## Basic script examples

**Current date/time:** `echo "Current date and time: $(date)"`  
**Disk usage:** `df -h`  
**List files:** `ls -l`  
**File exists:** `[ -f "$file" ] && echo "Exists" || echo "Not found"`  
**Add two numbers:** `sum=$((a + b))`  
**Uptime:** `uptime -p`  
**User exists:** `id "$user" &>/dev/null`  
**Backup dir:** `tar -czf backup_$(date +%F).tar.gz "$src"`  
**Internet check:** `ping -c 1 8.8.8.8 &>/dev/null`  
**String length:** `echo ${#str}`  

---

## Parameter expansion (Bash)

- `${str}` — Value of `str`.
- `${str^^}` — Uppercase (Bash 4+).
- `${str,,}` — Lowercase.
- `${str:pos:len}` — Substring.

**Note:** `${var^^}`, `[[ ]]`, arrays are **Bash-only**. Use `./script` (with shebang `#!/bin/bash`) or `bash script`; `sh script` may fail.

---

## Bash vs sh

| Case | Use |
|------|-----|
| Portable (cron, Docker entrypoint) | `#!/bin/sh` |
| Arrays, `[[ ]]`, `${var^^}`, etc. | `#!/bin/bash` |
| Minimal/embedded (e.g. Alpine) | `#!/bin/sh` |

---

## User creation script (standard format)

A robust pattern:

- Shebang and header comment (name, description, usage).
- `set -e` — exit on first error.
- Check root: `[[ $EUID -ne 0 ]]`.
- Check arguments: `[[ $# -ne 2 ]]`.
- Idempotent: check if user/group exists before creating.
- Use `getent group`, `id "$user"`.
- Optional: prompt for password, show `id` output.

---

## Assignment: Timestamped project directory

**Goal:** Reusable function + main script: create `/tmp/<project>-<YYYYMMDD-HHMMSS>` and print path.

**utils.sh:**

```bash
#!/bin/bash
create_timestamped_dir() {
    local project_name="$1"
    [ -z "$project_name" ] && echo "Error: Project name required." && return 1
    local timestamp=$(date +%Y%m%d-%H%M%S)
    local dir_path="/tmp/${project_name}-${timestamp}"
    mkdir -p "$dir_path"
    echo "Directory created: $dir_path"
}
```

**setup_project.sh:**

```bash
#!/bin/bash
source "$(dirname "$0")/utils.sh"
create_timestamped_dir "my-new-app"
```

Use `source "$(dirname "$0")/utils.sh"` so it works from any current directory.

---

# Part 5: Version Control with Git

## Why version control?

Without it: “project_final_v2_FINAL_updated” chaos. **Version control** tracks every change and who made it.

| Type | Description | Examples |
|------|-------------|----------|
| **Local** | Versions only on one machine | RCS |
| **Centralized** | One server; clients checkout | SVN, CVS |
| **Distributed** | Full copy on every machine | **Git**, Mercurial |

Git is distributed: every clone has full history; you can work offline and push later.

---

## Install and configure Git

```bash
# Install (example: RHEL/CentOS)
sudo yum update && sudo yum install git

# Ubuntu
sudo apt update -y && sudo apt install git -y

git --version
```

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
```

---

## SSH for GitHub

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Add the contents of `~/.ssh/id_rsa.pub` to GitHub: Settings → SSH and GPG keys.

---

## The four areas in Git

| Area | Also known as | Meaning |
|------|----------------|--------|
| **Working tree** | Workspace | Where you edit files |
| **Staging area** | Index | What will go in the next commit |
| **Local repository** | .git | Commits and history on your machine |
| **Remote** | origin | Hosted repo (e.g. GitHub) |

Flow: **Working tree** → `git add` → **Staging** → `git commit` → **Local repo** → `git push` → **Remote**.

---

## Essential Git commands

| Task | Command |
|------|---------|
| Init | `git init` |
| Status | `git status` |
| Stage | `git add <file>` or `git add .` |
| Commit | `git commit -m "message"` |
| Diff (working vs last commit) | `git diff` |
| Diff (staged vs last commit) | `git diff --cached` |
| Add remote | `git remote add origin <url>` |
| Push | `git push origin main` |
| Clone | `git clone <url>` |
| Pull | `git pull` |
| Help | `man git`, `git <cmd> -h` |

---

## Story summary: Maya’s Git journey

1. **Install & config** — Git, name, email, SSH key.
2. **GitHub** — Account, new repo (e.g. `demo-project`).
3. **Local start** — `mkdir demo && cd demo && git init`.
4. **First files** — `touch Abc.txt Xyz.txt` → `git status` (untracked).
5. **Stage** — `git add Abc.txt` → file in staging.
6. **First commit** — `git commit -m "Learn git commit"`.
7. **Remote** — `git remote add origin git@github.com:user/demo-project.git` then `git push origin main`.
8. **Later** — `git diff` (working), `git diff --cached` (staged).
9. **Collaboration** — Teammate runs `git clone <url>`, then branch, merge, push, pull.

**Lifecycle:** Working tree → `git add` → Staging → `git commit` → Local repo → `git push` → Remote. Updates: `git pull`.

---

*DevOps is a mindset, methodology, and toolset. Mastering these foundations improves delivery speed, quality, and collaboration.*
