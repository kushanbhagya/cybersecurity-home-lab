# 🔹 Lab 2: SSH Brute Force Attack & Log Analysis

## 🎯 Objective
The objective of this lab was to simulate an SSH brute force attack from Kali Linux against an Ubuntu server and detect the attack by analyzing authentication logs on the target machine.

---

## 🧱 Lab Environment

### 💻 Virtualization
- VMware Fusion

### 🖥️ Machines Used
- Kali Linux (Attacker)
- Ubuntu Server (Target)
- Windows 11 (Previously configured in the lab environment)

---

## 🌐 Network Details

| Machine | Role | IP Address |
|--------|------|-----------|
| Kali Linux | Attacker | 192.168.248.1 |
| Ubuntu Server | Target | 192.168.248.128 |

---

## 🛠️ Tools Used

- Hydra
- OpenSSH Server
- Ubuntu auth logs
- Kali Linux terminal

---

## 🚨 Attack Scenario Overview

In this lab, an SSH service was configured on the Ubuntu target machine.  
A test user account was created to simulate a login target.  
Hydra was then used from the Kali attacker machine to perform a brute force attack.  
Finally, the attack was detected by monitoring `/var/log/auth.log` on Ubuntu.

---

## 1️⃣ Install and Enable SSH on Ubuntu

SSH server was installed on the Ubuntu target machine.

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl start ssh
sudo systemctl enable ssh
sudo systemctl status ssh
```

### Result
- SSH installed successfully
- SSH service was active and running
- Port 22 was listening on the Ubuntu machine

### Screenshot
![SSH Setup](images/01-ssh-install-status.png)

---

## 2️⃣ Create Test User

A test user was created on Ubuntu for brute force simulation.

```bash
sudo adduser testuser
```

The password used for the test account was:

```text
password123
```

### Screenshot
![Create Test User](images/02-create-test-user.png)

---

## 3️⃣ Verify SSH Login from Kali

Before running Hydra, a normal SSH login was tested from Kali to Ubuntu.

```bash
ssh testuser@192.168.248.128
```

After accepting the SSH fingerprint and entering the correct password, login was successful.

### Result
- SSH access from Kali to Ubuntu worked correctly
- This confirmed that the service and credentials were valid

### Screenshot
![SSH Login Success](images/03-ssh-login-success.png)

---

## 4️⃣ First Hydra Attempt

The first brute force attempt was made using the default rockyou wordlist.

```bash
hydra -l testuser -P /usr/share/wordlists/rockyou.txt ssh://192.168.248.128
```

---

## ❌ Error 1: rockyou.txt Not Found

The attack failed because the password file did not exist in extracted form.

### Error Message

```text
[ERROR] File for passwords not found: /usr/share/wordlists/rockyou.txt
```

### Screenshot
![Hydra Rockyou Error](images/04-hydra-rockyou-error.png)

---

## ✅ Fix 1: Extract rockyou.txt.gz

The compressed wordlist was extracted using:

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
ls /usr/share/wordlists/
```

### Result
- `rockyou.txt` became available in the wordlists directory

### Screenshot
![Rockyou Extracted](images/05-rockyou-fix.png)

---

## 5️⃣ Second Hydra Attempt

After extracting the wordlist, Hydra was run again.

```bash
hydra -l testuser -P /usr/share/wordlists/rockyou.txt ssh://192.168.248.128
```

---

## ❌ Error 2: Too Many Connection Errors

The second attack attempt failed because SSH limited repeated parallel connections.

### Error Message

```text
[ERROR] all children were disabled due to too many connection errors
```

### Explanation
This happened because Hydra was making too many simultaneous SSH authentication attempts and the service could not handle them reliably.

### Screenshot
![Hydra Connection Error](images/06-hydra-connection-error.png)

---

## ✅ Fix 2: Reduce Threads and Add Wait Time

Hydra was retried with lower parallelism and wait time:

```bash
hydra -l testuser -P /usr/share/wordlists/rockyou.txt -t 4 -W 3 ssh://192.168.248.128
```

### Result
This still did not complete successfully in the lab environment.

---

## 6️⃣ Create Custom Password List

To make the test more controlled and reliable, a smaller custom password list was created.

```bash
nano passwords.txt
```

The following passwords were added:

```text
123456
admin
password
password123
test123
```

### Screenshot
![Custom Password List](images/07-custom-password-list.png)

---

## 7️⃣ Run Hydra with Custom Wordlist

Hydra was then run using the custom password list.

```bash
hydra -l testuser -P passwords.txt -t 2 ssh://192.168.248.128
```

### Result
- Hydra was able to run successfully using the smaller wordlist
- This made the brute force simulation more stable for the lab

### Screenshot
![Hydra Final Run](images/08-hydra-final-run.png)

---

## 8️⃣ Monitor Logs on Ubuntu

To detect the attack from the defender side, authentication logs were monitored on Ubuntu.

```bash
sudo tail -f /var/log/auth.log
```

### Screenshot
![Auth Log Monitoring](images/09-auth-log-detection.png)

---

## 🔍 Detection Results

The following attack evidence was observed in the authentication logs:

- Failed password attempts for user `testuser`
- Repeated SSH authentication failures
- Source attacker IP identified as `192.168.248.1`
- Connection attempts consistent with brute force behavior

Example observation:

```text
Failed password for testuser from 192.168.248.1
```

This confirmed that the brute force attack generated visible artifacts in system logs and could be detected through log analysis.

---

## 🧪 Attack Flow

1. Install and enable SSH on Ubuntu
2. Create a test user account
3. Verify SSH login manually from Kali
4. Launch Hydra brute force attack
5. Encounter missing wordlist error
6. Extract the rockyou wordlist
7. Encounter SSH connection limit issue
8. Reduce attack speed and retry
9. Create a custom password list
10. Run Hydra successfully with the custom list
11. Monitor `/var/log/auth.log`
12. Detect failed login attempts from attacker IP

---

## 🧠 Skills Demonstrated

- SSH service configuration
- User account setup in Linux
- Brute force attack simulation using Hydra
- Wordlist handling and preparation
- Log monitoring and analysis
- Troubleshooting connection-related issues
- Understanding attacker and defender perspectives

---

## 📘 Key Learnings

- SSH brute force attacks can be simulated safely in a lab environment
- Authentication logs provide strong visibility into failed login activity
- Large default wordlists may not always be practical in small lab setups
- Connection limits can affect brute force tools
- Creating a small custom wordlist is useful for controlled testing and demonstration

---

## ⚠️ Challenges and Fixes Summary

| Challenge | Cause | Fix |
|----------|------|-----|
| `rockyou.txt` not found | Wordlist was still compressed | Extracted `rockyou.txt.gz` |
| Too many connection errors | Hydra made too many parallel attempts | Reduced threads and added wait time |
| Hydra still unstable with large list | Lab environment limitations | Created and used a custom password list |

---

## 🚀 Conclusion

This lab successfully demonstrated an SSH brute force attack and showed how such activity can be identified through authentication logs on the target machine. It also highlighted common practical issues such as missing wordlists and connection throttling, along with the troubleshooting steps required to complete the exercise.

---

## ⚠️ Disclaimer

This lab was conducted in a controlled virtual environment for educational purposes only.