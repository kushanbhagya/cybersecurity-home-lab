# 🚨 Lab 8: Real-Time Log Alerting using Apache Logs

> *Build a custom Bash-based alerting system that monitors Apache access logs in real time — instantly flagging suspicious web activity and simulating a lightweight SOC detection workflow.*

---

## 📋 Table of Contents

- [Objective](#-objective)
- [Lab Environment](#-lab-environment)
- [Tools Used](#️-tools-used)
- [Scenario Overview](#-scenario-overview)
- [Lab Steps](#️-lab-steps)
  - [1. Create the Alert Script](#1️⃣-create-the-alert-script)
  - [2. Make Script Executable](#2️⃣-make-script-executable)
  - [3. Run the Alert Script](#3️⃣-run-the-alert-script)
  - [4. Launch Attack from Kali](#4️⃣-launch-attack-from-kali)
  - [5. Real-Time Detection](#5️⃣-real-time-detection)
- [Detection Logic](#-detection-logic)
- [Attack Flow](#-attack-flow)
- [Skills Demonstrated](#-skills-demonstrated)
- [Key Learnings](#-key-learnings)
- [Conclusion](#-conclusion)

---

## 🎯 Objective

Build a **custom Bash alerting script** that monitors Apache access logs in real time and generates instant terminal alerts when suspicious web activity is detected — simulating a basic **Security Operations Center (SOC)** monitoring workflow without requiring a full SIEM platform.

---

## 🧱 Lab Environment

| Setting | Details |
|---------|---------|
| Hypervisor | VMware Fusion |

### 🖥️ Machines

| Role | OS | IP Address |
|------|----|-----------|
| 🔴 Attacker | Kali Linux | `192.168.248.130` |
| 🟢 Defender | Ubuntu Server (Apache) | `192.168.248.128` |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **Apache2** | Web server generating access logs on Ubuntu |
| **Bash Script** (`alert.sh`) | Custom real-time alerting and detection engine |
| **Gobuster** | Web directory brute-force tool launched from Kali |
| **Apache Access Logs** | Data source monitored by the alert script |

---

## 🚨 Scenario Overview

```
[Kali Linux]
     │
     │  gobuster dir -u http://192.168.248.128
     ▼
[Ubuntu Server — Apache]
     │
     └── /var/log/apache2/access.log
                   │
              alert.sh (tail -Fn0)
                   │
         Pattern match: gobuster | .git
                   | .env | /admin | /login
                   │
         [ALERT] Suspicious activity detected ✅
         Attacker IP + request printed instantly
```

- A custom **Bash script** watches the Apache log file continuously using `tail -Fn0`
- Every new log line is scanned for suspicious patterns using `grep -E`
- When a match is found, an **instant alert** is printed to the terminal
- This simulates the core logic of a real-world **SOC detection rule**

---

## ⚙️ Lab Steps

---

### 1️⃣ Create the Alert Script

The alert script was created on the Ubuntu defender machine.

```bash
nano alert.sh
```

**Script contents (`alert.sh`):**

```bash
#!/bin/bash

LOG="/var/log/apache2/access.log"

echo "[+] Monitoring web attacks..."

tail -Fn0 $LOG | while read line
do
    echo "$line" | grep -E "gobuster|\.git|\.env|/admin|/login|/config" > /dev/null
    if [ $? = 0 ]; then
        echo "[ALERT] Suspicious activity detected:"
        echo "$line"
    fi
done
```

#### 🔍 Script Breakdown

| Component | Purpose |
|-----------|---------|
| `LOG=` | Path to the Apache access log being monitored |
| `tail -Fn0` | Follow the log file from the current end — picks up only new lines |
| `while read line` | Processes each new log entry as it arrives |
| `grep -E "..."` | Matches suspicious patterns using extended regex |
| `> /dev/null` | Suppresses grep output — only the `if` block fires on a match |
| `[ALERT]` | Prints a clear alert with the full matching log line |

#### 🎯 Patterns Being Detected

| Pattern | What It Catches |
|---------|----------------|
| `gobuster` | Gobuster tool identified via User-Agent |
| `\.git` | Attempts to access exposed Git repositories |
| `\.env` | Probing for environment/config files |
| `/admin` | Admin panel discovery attempts |
| `/login` | Login page enumeration |
| `/config` | Configuration file probing |

![Alert Script](images/01-alert-script.png)

---

### 2️⃣ Make Script Executable

The script was given execute permissions before running.

```bash
chmod +x alert.sh
```

---

### 3️⃣ Run the Alert Script

The alerting script was started on the Ubuntu defender **before** launching the attack — ensuring all incoming requests are monitored from the start.

```bash
./alert.sh
```

Expected startup output:
```
[+] Monitoring web attacks...
```

![Script Running](images/02-script-running.png)

---

### 4️⃣ Launch Attack from Kali

From the Kali Linux attacker machine, Gobuster was launched against the Apache web server.

```bash
gobuster dir -u http://192.168.248.128 -w /usr/share/wordlists/dirb/common.txt
```

![Gobuster Attack](images/03-gobuster-attack.png)

---

### 5️⃣ Real-Time Detection

The moment Gobuster's requests hit the Apache server, the alert script began generating live alerts in the terminal.

#### Example Alert Output

```
[+] Monitoring web attacks...
[ALERT] Suspicious activity detected:
192.168.248.130 - - [date] "GET /.git HTTP/1.1" 404 -
[ALERT] Suspicious activity detected:
192.168.248.130 - - [date] "GET /.env HTTP/1.1" 404 -
[ALERT] Suspicious activity detected:
192.168.248.130 - - [date] "GET /admin HTTP/1.1" 404 -
[ALERT] Suspicious activity detected:
192.168.248.130 - - [date] "GET /config HTTP/1.1" 404 -
```

![Real-Time Alerts](images/04-alert-triggered.png)

> ✅ Alerts fired **instantly** as each suspicious request arrived — no delay, no manual checking required.

---

## 🔍 Detection Logic

```
New line written to /var/log/apache2/access.log
              │
        tail -Fn0 reads it
              │
        grep -E checks for:
        ┌─────────────────────────────┐
        │  gobuster   → tool signature │
        │  .git       → repo exposure  │
        │  .env       → config leak    │
        │  /admin     → admin probing  │
        │  /login     → auth probing   │
        │  /config    → config probing │
        └─────────────────────────────┘
              │
        Match found?
        ├── YES → Print [ALERT] + full log line
        └── NO  → Continue silently
```

---

## 🧪 Attack Flow

```
 1. Create alert.sh on Ubuntu (detection script)
 2. chmod +x alert.sh
 3. Run ./alert.sh → starts monitoring access.log
 4. Launch Gobuster from Kali → attack begins
 5. Requests hit Apache → written to access.log
 6. alert.sh detects suspicious patterns ✅
 7. [ALERT] messages fire instantly in terminal ✅
```

---

## 🧠 Skills Demonstrated

- ✅ Writing a functional **Bash detection script** from scratch
- ✅ Real-time log file monitoring using `tail -Fn0`
- ✅ Pattern matching with `grep -E` for security use cases
- ✅ Identifying web attack signatures in Apache logs
- ✅ Simulating a **SOC alerting rule** without a SIEM
- ✅ Correlating attacker behavior across attack and defender tools
- ✅ Security automation through shell scripting

---

## 📘 Key Learnings

- ✅ Web enumeration attacks produce **consistent, detectable log patterns** — they can't hide in access logs
- ✅ `tail -Fn0` is the right flag for **live log following** — it only reads new lines, not old history
- ✅ A simple Bash script can replicate the **core logic of a SIEM detection rule**
- ✅ Real-time alerting dramatically reduces **mean time to detect (MTTD)**
- ✅ Lightweight custom scripts are valuable in environments **without enterprise security tooling**
- ✅ This script is a foundation that can be extended with **email alerts, IP blocking, or log file output**

---

## 🚀 Conclusion

This lab demonstrated how to build a real-time web attack alerting system using nothing more than a Bash script and Apache's built-in access logging. The system successfully detected Gobuster's directory brute-force attack the moment it started generating instant, actionable alerts that simulate the core behavior of a SOC monitoring workflow. This approach is directly scalable into more advanced detection pipelines using tools like **Splunk**, **ELK Stack**, or **Wazuh**.

---

## ⚠️ Disclaimer

> This lab was conducted in a **controlled virtual environment** for **educational purposes only**.  
> Do not replicate these techniques on any network or system without explicit written authorization.

---

<div align="center">

*🚨 Lab 8 — Real-Time Log Alerting with Bash · Cybersecurity Home Lab Series*

</div>
