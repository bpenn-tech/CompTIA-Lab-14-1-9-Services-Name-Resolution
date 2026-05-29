# 🔍 Lab 14.1.9 — Services/Name Resolution
A+ Core 1 lab: used nslookup to investigate DNS misconfiguration for corpnet.xyz and netstat to detect unauthorized VNC server on port 5900. 100% score.

> **CompTIA A+ Core 1 | CertMaster Perform v15 | Per Scholas AI-Enabled IT Support — NCR**

![Score](https://img.shields.io/badge/Score-100%25-brightgreen) ![Layer](https://img.shields.io/badge/OSI%20Layer-Application%20(L7)-purple) ![Status](https://img.shields.io/badge/Status-Complete-success) ![Program](https://img.shields.io/badge/Per%20Scholas-NCR-orange)

---

## 📋 Scenario

> *"As an IT technician, you've been asked to look into two separate network-related issues reported by your team."*

**Issue 1 — DNS Investigation:**
The team suspects a DNS configuration issue for the company's domain `corpnet.xyz`. Use `nslookup` to investigate and answer the first three questions.

**Issue 2 — Security Audit:**
A VNC server might be running on one of the computers. Go to the Support computer and use `netstat -l` to check for an active listener on port 5900.

---

## 🗺️ Network Layout

| Name | IP Address |
|:---|:---|
| Office1 | `192.168.0.30` |
| Office2 | `192.168.0.31` |
| Exec | `192.168.0.34` |
| Support | `199.92.0.32` |
| IT-Laptop | `192.168.0.33` |
| CorpServer | `192.168.0.10` |
| Internal DNS Server | `192.168.0.11` |

---

## 🖥️ Tools Used

| Tool | Machine | OS |
|:---|:---|:---|
| `nslookup` | IT-Laptop | Kali Linux |
| `netstat -l` | Support | Linux |

> 💭 **My thinking:** The lab dropped me into the IT Administration room with two machines — ITAdmin (Windows) and IT-Laptop (Linux/Kali). I initially ran commands on ITAdmin but discovered the Windows PowerShell version of nslookup had syntax conflicts with the `-type=mx` parameter. Switching to IT-Laptop's Linux terminal resolved the issue and all commands ran cleanly. **Lesson learned: always verify which machine and OS the lab intends before running commands.**

---

## 🔍 Part 1 — DNS Investigation with nslookup

### Step 1 — Query Primary IP for corpnet.xyz

```bash
nslookup corpnet.xyz
```

#### Output
Server: CorpDC.CorpNet.local
Address: 192.168.0.11
Non-authoritative answer:
Host Name: corpnet.xyz
Addresses: 198.28.1.1

**What it tells me:**
- Default internal DNS server is `CorpDC.CorpNet.local` at `192.168.0.11`
- Primary IP for `corpnet.xyz` = **`198.28.1.1`**
- "Non-authoritative" means the DNS server pulled this from cache — not its own records

> 💭 **My thinking:** This confirmed the internal DNS server IS responding and CAN resolve the domain to an IP. The non-authoritative flag is a subtle clue that this server may not be the primary authority for `corpnet.xyz` — it's getting the answer from somewhere else and caching it.

**Lab Question 1 answered ✅** — Primary IP = `198.28.1.1`

---

### Step 2 — Query MX Record from Internal DNS

```bash
nslookup -type=mx corpnet.xyz
```

#### Output
Server: CorpDC.CorpNet.local
Address: 192.168.0.11
Non-authoritative answer:
corpnet.xyz mail exchanger = 5 www3.corpnet.xyz

**What it tells me:**
- The internal DNS DOES have an MX record for `corpnet.xyz`
- Mail server name = `www3.corpnet.xyz`
- The `5` is the **priority number** — lower = higher priority

> 💭 **My thinking:** This worked on Linux but timed out on Windows ITAdmin — confirming the OS matters for nslookup syntax. The internal DNS returned `www3.corpnet.xyz` as the mail server. I noted this and compared it with what the external DNS would return in the next step.

---

### Step 3 — Query Mail Server IP

```bash
nslookup www3.corpnet.xyz
```

#### Output
Server: CorpDC.CorpNet.local
Address: 192.168.0.11
Non-authoritative answer:
Host Name: www3.corpnet.xyz
Addresses: 198.28.1.3

**What it tells me:**
- IP address of the mail server `www3.corpnet.xyz` = **`198.28.1.3`**

> 💭 **My thinking:** Now I had the full picture for the internal DNS — domain resolves to `198.28.1.1` and the mail server is at `198.28.1.3`. Both are on the `198.28.1.x` subnet which makes sense for a corporate mail infrastructure.

**Lab Question 2 answered ✅** — Mail server IP = `198.28.1.3`

---

### Step 4 — Query MX Record from External DNS Server

```bash
nslookup -type=mx corpnet.xyz ns1.nethost.net
```

#### Output
Server: ns1
Address: 163.128.80.93
Non-authoritative answer:
corpnet.xyz mail exchanger = 5 corpnet-www3.corpnet.xyz

**What it tells me:**
- External DNS server `ns1.nethost.net` is at `163.128.80.93`
- External DNS returns mail server name = **`corpnet-www3.corpnet.xyz`**
- This is DIFFERENT from what the internal DNS returned (`www3.corpnet.xyz`)

> 💭 **My thinking:** This is the DNS misconfiguration the lab was investigating. The internal DNS says the mail server is `www3.corpnet.xyz` but the external DNS says it's `corpnet-www3.corpnet.xyz`. These don't match — meaning external senders trying to deliver email to `corpnet.xyz` would be directed to a different server than internal users. In a real job I would flag this to the DNS administrator to reconcile the internal and external records.

**Lab Question 3 answered ✅** — External DNS server name = `corpnet-www3.corpnet.xyz`

---

## 🔐 Part 2 — Security Audit with netstat

### Step 5 — Check for Active Listeners on Port 5900

Navigated to **Support Office → Support computer** and ran:

```bash
netstat -l
```

#### Relevant Output
Active Internet connections (only servers)
Proto  Local Address      State
tcp    0.0.0.0:5901       LISTEN
tcp    0.0.0.0:5900       LISTEN

**What it tells me:**
- Port `5900` is actively **LISTEN**ing — VNC server is running ✅
- Port `5901` is also listening — second VNC display is active
- Both ports accepting connections from any IP (`0.0.0.0`)

> 💭 **My thinking:** Finding port 5900 listening confirmed an active VNC server on the Support computer. VNC (Virtual Network Computing) is a remote desktop protocol — if nobody authorized this installation it represents a serious security vulnerability. Anyone who can reach this machine on port 5900 could potentially take full remote control of it. This is exactly what security audits look for — unauthorized services running on network machines.
>
> **Where I got stuck:** I initially typed `netstat -1` (number one) instead of `netstat -l` (lowercase L). They look almost identical but mean completely different things. The `-l` flag means "listening" — always double check lowercase L vs number 1 in terminal commands.

**Lab Question 4 answered ✅** — Active listener on port 5900 = **Yes**

---

## 🎯 Root Cause Summary

### Issue 1 — DNS Misconfiguration

| | Internal DNS | External DNS |
|:---|:---|:---|
| **Server** | `CorpDC.CorpNet.local` | `ns1.nethost.net` |
| **Address** | `192.168.0.11` | `163.128.80.93` |
| **MX Record** | `www3.corpnet.xyz` | `corpnet-www3.corpnet.xyz` |
| **Match?** | ❌ Records don't match | |

**Impact:** Internal and external mail routing for `corpnet.xyz` point to different servers — potential email delivery failures for external senders.

### Issue 2 — Unauthorized VNC Server

| | |
|:---|:---|
| **Machine** | Support computer |
| **Port** | 5900 (and 5901) |
| **Protocol** | VNC — Virtual Network Computing |
| **Status** | LISTEN — accepting connections |
| **Risk** | Unauthorized remote desktop access |
| **Recommended action** | Investigate who installed VNC, disable if unauthorized |

---

## 🧠 Key Concepts Demonstrated

| Concept | Applied |
|:---|:---|
| DNS record types | A record, MX record queried with nslookup |
| Internal vs external DNS | Identified misconfiguration by comparing results |
| nslookup -type parameter | Pulled specific record types on demand |
| MX record priority | Understood the `5` priority value in MX results |
| Non-authoritative answers | Recognized cached DNS responses |
| netstat -l | Identified active listening ports |
| Port 5900 / VNC | Confirmed unauthorized remote desktop service |
| Linux vs Windows CLI | Discovered OS-specific syntax differences for nslookup |
| Security audit methodology | Checked for unauthorized services on network machines |

---

## 📸 Screenshot Log

| # | File | What it shows |
|:---|:---|:---|
| 01 | `01-lab-instructions.png` | Lab scenario and both issues |
| 02 | `02-itadmin-desktop-lab-questions.png` | Lab questions panel — all 4 questions |
| 03 | `03-floor-plan-overview.png` | Corporate floor layout |
| 04 | `04-itlaptop-nslookup-primary-ip.png` | corpnet.xyz = 198.28.1.1 |
| 05 | `05-itlaptop-nslookup-mx-internal.png` | Internal MX = www3.corpnet.xyz |
| 06 | `06-itlaptop-nslookup-mailserver-ip.png` | Mail server IP = 198.28.1.3 |
| 07 | `07-itlaptop-nslookup-mx-external.png` | External MX = corpnet-www3.corpnet.xyz |
| 08 | `08-support-netstat-port5900.png` | Port 5900 LISTEN — VNC confirmed |
| 09 | `09-lab-questions-corrected.png` | All 4 answers filled in correctly |
| 10 | `10-lab-score-100-percent.png` | Final score: 100% |

---

## 🛠️ Tools & Technologies

`nslookup` &nbsp;|&nbsp; `netstat -l` &nbsp;|&nbsp; `Kali Linux Terminal` &nbsp;|&nbsp; `DNS Records (A, MX)` &nbsp;|&nbsp; `VNC / Port 5900` &nbsp;|&nbsp; `CompTIA CertMaster Perform`

---

## 📚 CompTIA A+ Exam Objectives Covered

**220-1101 Core 1**
- `2.1` — Compare and contrast TCP and UDP ports, protocols, and their purposes
- `2.6` — Compare and contrast common network configuration concepts (DNS)
- `2.8` — Given a scenario, use networking tools (`nslookup`, `netstat`)

---

## 💡 What I Learned

This lab taught me that DNS is not a single monolithic system — internal and external DNS servers can have completely different records for the same domain, and those differences can cause real-world problems like misrouted email. Comparing results between internal and external DNS servers is a diagnostic technique I'll carry into real helpdesk work.

The netstat discovery reinforced something important: security is part of every IT support role. An unauthorized VNC server is exactly the kind of thing that gets missed until someone looks — and `netstat -l` is how you look.

---

*Part of an ongoing IT support portfolio built during the Per Scholas AI-Enabled IT Support — National Capital Region program.*
