# 🔐 Lab 4: Preventing SSH Brute Force Attacks using Fail2Ban

> *Detect and automatically block SSH brute force attacks by configuring Fail2Ban to monitor authentication logs and ban malicious IPs after repeated failed login attempts.*

---

## 📋 Table of Contents

- [Objective](#-objective)
- [Lab Environment](#-lab-environment)
- [Tools Used](#️-tools-used)
- [Scenario Overview](#-scenario-overview)
- [Lab Steps](#️-lab-steps)
  - [1. Install Fail2Ban](#1️⃣-install-fail2ban)
  - [2. Verify Service Status](#2️⃣-verify-service-status)
  - [3. Configure Fail2Ban](#3️⃣-configure-fail2ban)
  - [4. Restart Fail2Ban](#4️⃣-restart-fail2ban)
  - [5. Verify Jail Status](#5️⃣-verify-jail-status)
  - [6. Simulate Brute Force Attack](#6️⃣-simulate-brute-force-attack)
  - [7. Force Failed Attempts](#7️⃣-fix-force-failed-attempts)
  - [8. Monitor Logs in Real Time](#8️⃣-monitor-logs-real-time-detection)
  - [9. Confirm Attacker Blocked](#9️⃣-confirm-attacker-blocked)
  - [10. Test SSH After Ban](#-test-ssh-after-ban)
- [Key Observations](#-key-observations)
- [Important Insight](#️-important-insight)
- [Attack Flow](#-attack-flow)
- [Skills Demonstrated](#-skills-demonstrated)
- [Key Learnings](#-key-learnings)
- [Conclusion](#-conclusion)

---

## 🎯 Objective

Configure **Fail2Ban** on an Ubuntu Server to automatically detect and block SSH brute force attacks banning the attacker's IP after a defined number of failed login attempts, and verifying the block from the Kali Linux attacker machine.

---

## 🧱 Lab Environment

| Setting | Details |
|---------|---------|
| Hypervisor | VMware Fusion |

### 🖥️ Machines

| Role | OS | IP Address |
|------|----|-----------|
| 🔴 Attacker | Kali Linux | `192.168.248.130` |
| 🟢 Target | Ubuntu Server | `192.168.248.128` |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **Fail2Ban** | Monitor logs and auto-ban malicious IPs |
| **Hydra** | Simulate SSH brute force attack |
| **OpenSSH** | Target SSH service on Ubuntu |
| **`/var/log/auth.log`** | Authentication log monitored by Fail2Ban |

---

## 🚨 Scenario Overview

```
[Kali Linux] ──── Hydra SSH Brute Force ────▶ [Ubuntu Server]
                                                      │
                                              /var/log/auth.log
                                                      │
                                                  Fail2Ban
                                                      │
                                          Detects repeated failures
                                                      │
                                          iptables ban on attacker IP
                                                      │
                  Connection refused ◀────────────────┘
```

- Hydra launched from **Kali Linux** to brute force SSH on **Ubuntu**
- **Fail2Ban** monitors `auth.log` for repeated failures
- After `maxretry` threshold hit, attacker IP is **automatically banned**
- Further SSH attempts from Kali are **blocked at the firewall level**

---

## ⚙️ Lab Steps

---

### 1️⃣ Install Fail2Ban

Fail2Ban was installed on the Ubuntu Server.

```bash
sudo apt update
sudo apt install fail2ban -y
```

![Fail2Ban Install](images/01-install-fail2ban.png)

---

### 2️⃣ Verify Service Status

Confirmed Fail2Ban started correctly after installation.

```bash
sudo systemctl status fail2ban
```

Expected output:
```
● fail2ban.service - Fail2Ban Service
     Active: active (running)
```

![Fail2Ban Status](images/02-fail2ban-status.png)

---

### 3️⃣ Configure Fail2Ban

#### Step 1 — Copy default config to local override:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

> 💡 Always edit `jail.local` — never modify `jail.conf` directly, as it may be overwritten on updates.

#### Step 2 — Open the config file:

```bash
sudo nano /etc/fail2ban/jail.local
```

#### Step 3 — Configure the `[sshd]` jail:

```ini
[sshd]
enabled  = true
port     = ssh
logpath  = /var/log/auth.log
backend  = %(sshd_backend)s
maxretry = 3
bantime  = 600
findtime = 600
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `maxretry` | `3` | Ban after 3 failed attempts |
| `bantime` | `600` | Ban lasts 600 seconds (10 min) |
| `findtime` | `600` | Count failures within 600-second window |

![Fail2Ban Config](images/03-fail2ban-config.png)

---

### 4️⃣ Restart Fail2Ban

Applied the new configuration by restarting the service.

```bash
sudo systemctl restart fail2ban
```

---

### 5️⃣ Verify Jail Status

Confirmed the SSH jail was active and monitoring.

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

![Jail Status](images/04-jail-status.png)

---

### 6️⃣ Simulate Brute Force Attack

The initial attack was launched from Kali Linux using Hydra with a password list that included the correct password.

```bash
hydra -l testuser -P passwords.txt -t 2 ssh://192.168.248.128
```

#### ⚠️ Observation — No Ban Triggered

| Issue | Explanation |
|-------|-------------|
| Attack succeeded immediately | The correct password was found before 3 failures |
| Fail2Ban did **not** ban the IP | No ban is triggered if the threshold isn't reached |

> 🧠 **Key Insight:** Fail2Ban only triggers on **failed** attempts. A successful login before the threshold is hit will not result in a ban.

---

### 7️⃣ Fix: Force Failed Attempts

To properly trigger Fail2Ban, a new wordlist containing **only wrong passwords** was created.

```bash
nano wrong_passwords.txt
```

```text
123456
password
admin
test
qwerty
abc123
```

#### Re-run the attack with the wrong password list:

```bash
hydra -l testuser -P wrong_passwords.txt -t 2 ssh://192.168.248.128
```

![Hydra Attack](images/05-hydra-attack.png)

---

### 8️⃣ Monitor Logs: Real-Time Detection

Authentication logs were monitored on Ubuntu to observe Fail2Ban reacting to the attack.

```bash
sudo tail -f /var/log/auth.log
```

#### Observed in logs:
- Repeated `Failed password for testuser` entries from `192.168.248.130`
- Fail2Ban detected the pattern and triggered a ban

![Auth Log](images/06-auth-log.png)

---

### 9️⃣ Confirm Attacker Blocked

The Fail2Ban jail status was checked to confirm the attacker's IP was added to the ban list.

```bash
sudo fail2ban-client status sshd
```

Expected output:
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: X
|  |- Total failed:     X
|  `- File list:        /var/log/auth.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   192.168.248.130
```

![Banned IP](images/07-banned-ip.png)

---

### 🔴 Test SSH After Ban

An SSH connection was attempted from Kali Linux to confirm the block was effective.

```bash
ssh testuser@192.168.248.128
```

#### ✅ Result
- Connection **refused** or **timed out**
- Access successfully blocked at the firewall level

![SSH Blocked](images/08-ssh-blocked.png)

---

## 🔍 Key Observations

| Observation | Detail |
|-------------|--------|
| Log monitored | `/var/log/auth.log` |
| Trigger condition | 3+ failed SSH attempts within 600 seconds |
| Action taken | Attacker IP banned via `iptables` |
| Ban duration | 600 seconds (10 minutes) |
| Attacker IP banned | `192.168.248.130` |
| Post-ban SSH result | Connection refused / timeout |

---

## ⚠️ Important Insight

> **Fail2Ban only triggers when the failure threshold is reached within the time window.**
>
> If the correct password is found before `maxretry` failures occur, no ban is issued, highlighting why **strong, unique passwords** and **key-based authentication** are essential complements to Fail2Ban.

---

## 🧪 Attack Flow

```
 1. Install & configure Fail2Ban on Ubuntu
 2. Set [sshd] jail: maxretry=3, bantime=600
 3. Launch Hydra from Kali with correct password list
        └── ⚠️  Attack succeeds — no ban triggered
        └── 🧠  Reason: correct password found too early
 4. Create wrong_passwords.txt (no correct password)
 5. Re-run Hydra — all attempts fail
 6. Fail2Ban detects 3+ failures within findtime
 7. Attacker IP (192.168.248.130) automatically banned ✅
 8. SSH from Kali → Connection refused ✅
```

---

## 🧠 Skills Demonstrated

- ✅ Installing and configuring Fail2Ban on Linux
- ✅ Understanding jail configuration parameters
- ✅ Simulating and adjusting brute force attacks
- ✅ Real-time log monitoring and analysis
- ✅ Verifying automated IP banning
- ✅ Testing firewall-level blocks from the attacker side
- ✅ Defensive security configuration and hardening

---

## 📘 Key Learnings

- ✅ Fail2Ban is highly effective against automated brute force attacks
- ✅ Proper configuration of `maxretry`, `bantime`, and `findtime` is critical
- ✅ The tool relies on **failed** attempts a successful early login bypasses it
- ✅ Logs are the foundation of automated detection; keeping them clean matters
- ✅ Fail2Ban should be combined with strong passwords and SSH key auth for full protection

---

## 🚀 Conclusion

This lab demonstrated how **Fail2Ban** can be configured to automatically detect and block SSH brute force attacks in real time. By combining log analysis, behavioral thresholds, and automated firewall rules, Fail2Ban provides a practical and effective first line of defense, replicating defensive techniques used in real-world production systems.

---

## ⚠️ Disclaimer

> This lab was conducted in a **controlled virtual environment** for **educational purposes only**.  
> Do not replicate these techniques on any network or system without explicit written authorization.

---

<div align="center">

*🔐 Lab 4 — SSH Brute Force Prevention with Fail2Ban · Cybersecurity Home Lab Series*

</div>
