# 🌐 Lab 7: Web Attack Detection using Apache Logs

> *Simulate a web directory brute-force attack using Gobuster, then detect and analyze the attack in real time by monitoring Apache access logs, identifying the attacker, their tools, and their targets.*

---

## 📋 Table of Contents

- [Objective](#-objective)
- [Lab Environment](#-lab-environment)
- [Tools Used](#️-tools-used)
- [Scenario Overview](#-scenario-overview)
- [Lab Steps](#️-lab-steps)
  - [0. Verify Apache Service](#-verify-apache-service)
  - [1. Start Log Monitoring](#1️⃣-start-log-monitoring)
  - [2. Launch Attack from Kali](#2️⃣-launch-attack-from-kali)
  - [3. Detect Attack in Logs](#3️⃣-detect-attack-in-logs)
- [Attack Indicators](#-attack-indicators)
- [Identify the Attacker](#️-identify-the-attacker)
- [Example Log Entries](#-example-log-entries)
- [Analysis & Security Impact](#-analysis--security-impact)
- [Skills Demonstrated](#-skills-demonstrated)
- [Key Learnings](#-key-learnings)
- [Conclusion](#-conclusion)

---

## 🎯 Objective

Detect a **web directory brute-force attack** by analyzing Apache access logs in real time. Using Gobuster as the attack tool from Kali Linux, the lab demonstrates how automated web enumeration leaves clear, identifiable patterns in server logs — and how defenders can use those patterns to detect and attribute attacks.

---

## 🧱 Lab Environment

| Setting | Details |
|---------|---------|
| Hypervisor | VMware Fusion |

### 🖥️ Machines

| Role | OS | IP Address |
|------|----|-----------|
| 🔴 Attacker | Kali Linux | `192.168.248.130` |
| 🟢 Target | Ubuntu Server (Apache) | `192.168.248.128` |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **Apache2** | Web server running on Ubuntu — generates access logs |
| **Gobuster** | Directory brute-force tool run from Kali |
| **`tail -f`** | Real-time log monitoring on the defender side |

---

## 🚨 Scenario Overview

```
[Kali Linux]
     │
     │  gobuster dir -u http://192.168.248.128
     │  (wordlist: dirb/common.txt)
     ▼
[Ubuntu Server — Apache]
     │
     ├── Rapid HTTP GET requests to hundreds of paths
     ├── 404s for missing paths
     ├── 403s for restricted paths
     │
     └── /var/log/apache2/access.log
              │
         Real-time detection via tail -f
              │
         Attacker IP identified ✅
         Tool fingerprinted ✅
         Targets revealed ✅
```

- **Gobuster** fires hundreds of HTTP requests per second to guess hidden directories
- **Apache** logs every request — including those that return 404 and 403
- **`tail -f`** watches the log file live, surfacing the attack as it happens

---

## ⚙️ Lab Steps

---

### 0️⃣ Verify Apache Service

Before starting the attack simulation, Apache was confirmed to be running on the Ubuntu target.

```bash
sudo systemctl status apache2
```

![Apache Status](images/00-apache-test.png)

---

### 1️⃣ Start Log Monitoring

Apache's access log was opened for real-time monitoring on the defender side **before** the attack was launched — simulating active log watching.

```bash
sudo tail -f /var/log/apache2/access.log
```

> 💡 The `-f` flag follows the file as new lines are written, making it ideal for live attack detection.

![Log Monitoring](images/01-log-monitoring.png)

---

### 2️⃣ Launch Attack from Kali

From the Kali Linux attacker machine, Gobuster was run against the Apache web server using a common directory wordlist.

```bash
gobuster dir -u http://192.168.248.128 -w /usr/share/wordlists/dirb/common.txt
```

| Flag | Value | Purpose |
|------|-------|---------|
| `dir` | — | Directory/file brute-force mode |
| `-u` | `http://192.168.248.128` | Target URL |
| `-w` | `dirb/common.txt` | Wordlist of common directory/file names |

![Gobuster Attack](images/02-gobuster-attack.png)

---

### 3️⃣ Detect Attack in Logs

While the attack was running, Apache's access log showed a rapid flood of requests — instantly visible in the `tail -f` output.

![Attack in Logs](images/03-log-detection.png)

---

## 🚨 Attack Indicators

The following patterns in the Apache logs confirm a directory brute-force attack:

| Indicator | Observed Behavior |
|-----------|------------------|
| 🚩 **High request rate** | Hundreds of requests per second — far beyond normal browsing |
| 🚩 **Unknown paths** | Requests for non-existent files and directories |
| 🚩 **Repeated 404 responses** | Server returning "Not Found" for enumerated paths |
| 🚩 **403 responses** | Server blocking access to restricted paths — but attacker is probing them |
| 🚩 **User-Agent fingerprint** | `gobuster/3.8` visible in log — tool identity exposed |
| 🚩 **Single source IP** | All requests originating from `192.168.248.130` |

---

## 🕵️ Identify the Attacker

From the Apache access logs, the attacker was immediately attributed:

| Field | Value |
|-------|-------|
| Source IP | `192.168.248.130` 🔴 (Kali Linux) |
| Tool identified | `gobuster/3.8` (visible in User-Agent) |
| Attack type | Directory enumeration / brute-force |
| Target server | `192.168.248.128` (Ubuntu — Apache) |

> 🧠 **Key Insight:** Many attack tools expose themselves through their User-Agent string. Gobuster, Nikto, sqlmap, and others all leave distinctive fingerprints in web server logs.

---

## 📌 Example Log Entries

The following is a sample of what was observed in `/var/log/apache2/access.log` during the attack:

```log
192.168.248.130 - - [date] "GET /.config HTTP/1.1"       404 -
192.168.248.130 - - [date] "GET /.git/HEAD HTTP/1.1"     404 -
192.168.248.130 - - [date] "GET /.bash_history HTTP/1.1" 404 -
192.168.248.130 - - [date] "GET /.ssh HTTP/1.1"          404 -
192.168.248.130 - - [date] "GET /.htaccess HTTP/1.1"     403 -
```

| Path Probed | HTTP Code | Significance |
|-------------|-----------|-------------|
| `/.config` | `404` | Probing for config files |
| `/.git/HEAD` | `404` | Checking for exposed Git repository |
| `/.bash_history` | `404` | Attempting to read shell history |
| `/.ssh` | `404` | Looking for SSH keys |
| `/.htaccess` | `403` | File exists but access denied — noted by attacker |

> ⚠️ A `403` response is more dangerous than `404` — it tells the attacker the path **exists**, just restricted. This is still valuable intelligence for an attacker.

---

## 🔍 Analysis & Security Impact

### What this attack demonstrates:
- **Directory brute-force scanning** — automated guessing of file and folder names
- **Sensitive file enumeration** — targeting config files, credentials, and admin panels
- **Reconnaissance before exploitation** — this is often a precursor to a deeper attack

### If successful, an attacker could discover:

| Target | Risk |
|--------|------|
| Config files | Database credentials, API keys |
| Backup files | Full copies of source code or databases |
| `.git` directory | Entire version history and secrets |
| Hidden admin panels | Unauthenticated access to admin interfaces |
| `.htaccess` | Server configuration and access rules |

---

## 🧠 Skills Demonstrated

- ✅ Verifying and managing web server services
- ✅ Real-time log monitoring with `tail -f`
- ✅ Recognizing automated web attack patterns in logs
- ✅ Fingerprinting attack tools via User-Agent strings
- ✅ Attacker IP attribution from access logs
- ✅ Understanding HTTP response codes as attack intelligence
- ✅ Analyzing the security implications of exposed paths

---

## 📘 Key Learnings

- ✅ Apache access logs record **every request** — including failed ones, which reveal attack intent
- ✅ Automated tools like Gobuster have **recognizable patterns**: high rate, sequential paths, consistent User-Agent
- ✅ A **403 response** is more informative to an attacker than a 404 — it confirms the path exists
- ✅ **Real-time log monitoring** with `tail -f` is a simple but powerful detection technique
- ✅ Attacker tools often **self-identify** through User-Agent strings — always check this field
- ✅ Web attack detection is the foundation for building **Web Application Firewall (WAF)** rules

---

## 🚀 Conclusion

This lab demonstrated how web directory brute-force attacks leave clear and distinctive traces in Apache access logs. By monitoring logs in real time, the attack was detected immediately — revealing the attacker's IP, their tool, the paths they targeted, and the server's responses. These skills form the foundation of web application security monitoring and are directly applicable to configuring WAFs, SIEM alerting rules, and incident response workflows.

---

## ⚠️ Disclaimer

> This lab was conducted in a **controlled virtual environment** for **educational purposes only**.  
> Do not replicate these techniques on any network or system without explicit written authorization.

---

<div align="center">

*🌐 Lab 7 — Web Attack Detection with Apache Logs · Cybersecurity Home Lab Series*

</div>
