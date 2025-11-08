# SSH Brute-force Detection — Project Documentation

## 🛡️ Important (Ethics & Safety)
This project demonstrates how to **detect, analyze, and respond to SSH brute-force behavior** in a **safe, controlled lab environment**.  
It does **not** provide instructions to attack systems.  
Do **not** run offensive actions against systems you do not own or have explicit written permission to test.  

For demonstration, this project uses **simulated SSH log events** or attacks performed **only within your own lab VM**.

---

## 1. Project Overview

### 🎯 Goal
Build a small lab that demonstrates the full defensive lifecycle of SSH log analysis and brute-force detection using **Splunk SIEM**.

### Steps Overview
1. Connect to the Kali VM via **PuTTY** (for demo only).  
2. Generate simulated SSH failed/accepted login events on Kali.  
3. Collect SSH logs and prepare them for ingestion.  
4. Transfer the log file to the **Splunk** host (Windows).  
5. Index logs in Splunk, write **SPL searches** to detect brute-force attempts, and create a simple dashboard.  
6. Demonstrate basic response: **fail2ban configuration** and manual IP blocking examples.

---

## 2. Lab Environment & Prerequisites

### 🖥️ Host/VMs
- **Kali Linux VM** (target for SSH logs) — owned by you.  
- **Windows system** running **Splunk Enterprise/Free** — for analysis and visualization.  
- Both on the same LAN (use **WinSCP**, **PSCP**, or **shared folder** for file transfer).

### 🧰 Software
- **PuTTY** — for SSH demonstration.  
- **WinSCP / PSCP** — for file transfer.  
- **Python 3** (on Kali) — for optional log simulation.  
- **Splunk** (on Windows).  
- **fail2ban** (optional) — for automated IP banning.

### 👥 Accounts
- Non-privileged user on Kali (e.g., `killerpk`).  
- Splunk admin account on Windows.

---

## 3. Ethical Statement
✅ No external systems are attacked.  
✅ Only simulated SSH logs or internal VM attacks are demonstrated.  
✅ This ensures compliance with ethical hacking and cybersecurity research standards.

---

## 4. PuTTY — Connecting to Kali (Demo Steps)

1. Open **PuTTY** on Windows.  
2. Enter **Host Name/IP:** `192.168.1.50` (your Kali VM).  
3. Port: `22`, Connection type: `SSH`.  
4. Click **Open** and log in with your Kali user credentials.

### 🔐 Simulating a Failed Login (Safe Demo)
1. Open PuTTY → connect to your Kali IP.  
2. At the password prompt, type a **wrong password** once → press Enter.  
3. Observe the “Access denied” message.  
4. Close and reconnect with the correct password.

---

## 5. Export SSH Logs from Kali

```bash
# Export SSH service logs from the last 15 minutes
sudo journalctl -u ssh --since "15 minutes ago" > ~/ssh_demo_recent.txt

# Verify last 50 lines
tail -n 50 ~/ssh_demo_recent.txt

# Compress (optional)
gzip -c ~/ssh_demo_recent.txt > ~/ssh_demo_recent.txt.gz
7. Upload the File to Splunk

Open Splunk Web → http://localhost:8000

Go to Settings → Add Data → Upload

Choose the file: ssh_demo_recent.txt

Set:

Source type: sshlogs (or syslog)

Index: main or kali_logs

Click Next → Review → Submit

8. Verify & Analyze Logs with SPL Queries
A — Show Failed and Accepted Logins
index=kali_logs sourcetype=sshlogs ("Failed password" OR "Accepted password")
| table _time host user src_ip port _raw
| sort 0 _time

B — Recent Failed Attempts
index=kali_logs sourcetype=sshlogs "Failed password" earliest=-1h
| rex "from (?<src_ip>\\d+\\.\\d+\\.\\d+\\.\\d+)"
| stats count AS failures by src_ip, user
| sort - failures

C — Accepted Logins Only
index=kali_logs sourcetype=sshlogs "Accepted password" earliest=-1h
| table _time host user src_ip port _raw

D — Failed vs Accepted Timeline
index=kali_logs sourcetype=sshlogs ("Failed password" OR "Accepted password") earliest=-1h
| rex "for (?<user>\\S+) from (?<src_ip>\\d+\\.\\d+\\.\\d+\\.\\d+) port (?<port>\\d+)"
| eval status = if(match(_raw,"(?i)Failed password"),"failed","accepted")
| timechart span=1m count by status
