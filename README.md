# All in one CTF (Tryhackme)

Target IP: 10.66.191.155\
Difficulty: Easy&#x20;

Link of room:[https://tryhackme.com/room/allinonemj](https://tryhackme.com/room/allinonemj)\
Author: Oussama Zehri \
Category: Web Exploitation, Cryptography, Privilege Escalation

{% stepper %}
{% step %}
### Phase I: Reconnaissance & Enumeration

Every successful breach starts with solid intel. Since this was a CTF environment, I opted for an aggressive Nmap scan to quickly map the attack surface.

{% code title="nmap_scan.txt" %}
```bash
nmap -T4 -sCV -A 10.66.191.155 -oN nmap_scan.txt
```
{% endcode %}

Open Ports & Services

| Port | Service | Version       | Notes                    |
| ---: | ------- | ------------- | ------------------------ |
|   21 | FTP     | vsftpd 3.0.5  | Anonymous Login Enabled. |
|   22 | SSH     | OpenSSH 8.2p1 | Standard Ubuntu SSH.     |
|   80 | HTTP    | Apache 2.4.41 | Hosting a WordPress CMS. |

{% hint style="info" %}
Red Team Note: While Anonymous FTP was open, a manual inspection yielded no sensitive files. I immediately pivoted to the Web entry point.
{% endhint %}
{% endstep %}

{% step %}
### Phase II: Web Discovery

I initiated a directory brute-force using Gobuster to find hidden endpoints.

{% code title="gobuster" %}
```bash
gobuster dir -u http://10.66.191.155/ -w /usr/share/wordlists/dirb/common.txt
```
{% endcode %}

Key Findings:

* /wordpress/: A standard WP installation.
* /hackathons/: A custom page containing interesting source code comments.

#### The Hidden Breadcrumb

Inspecting the source code of /hackathons/ revealed a cryptic comment:\
and.\
This pattern suggested a Vigenère Cipher. In real-world engagements, developers often leave "reminders" in comments that serve as the keys to the kingdom.
{% endstep %}

{% step %}
### Phase III: Initial Access

#### Cracking the Vigenère

Using the clues found on the site, I decrypted the string. The result provided the administrative credentials needed for the WordPress dashboard.

#### Exploiting WordPress (RCE)

With Admin access to WordPress, the path to a shell is straightforward via the Theme Editor:

* Navigate to Appearance > Theme Editor.
* Select `functions.php`.
* Inject a PHP reverse shell payload.
* Set up a listener: `nc -nvlp 4444`.

Callback Received:

```bash
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
{% endstep %}

{% step %}
### Phase IV: Lateral Movement

A www-data shell is restricted. I searched for files associated with the user `elyana`:

```bash
find / -user elyana 2>/dev/null
```

Discovery: `/etc/mysql/conf.d/private.txt`\
This file contained the plaintext password for `elyana`. Using this, I upgraded my session:

* `sudo su elyana` (or via SSH using the recovered credentials)
{% endstep %}

{% step %}
### Phase V: Privilege Escalation (Root)

To reach the top of the food chain, I checked for sudo misconfigurations:

```bash
sudo -l
```

#### The Vulnerability

`elyana` was permitted to run `/usr/bin/socat` as root without a password.

#### The Exploit (GTFOBins Style)

Socat is a powerful networking tool that can be abused to execute a shell with elevated privileges.

```bash
sudo /usr/bin/socat stdin exec:/bin/sh
```

Outcome:

```bash
id
uid=0(root) gid=0(root) groups=0(root)
```

System Compromised.
{% endstep %}

{% step %}
### Phase VI: Post-Mortem & Remediation

#### What I Learned

* Vigenère Awareness: Cryptographic obfuscation in CTFs often relies on classic ciphers. Recognizing these patterns saves hours of fruitless brute-forcing.
* The Power of Sudo: A single binary like `socat` can render all other security layers useless if sudo permissions are too broad.

#### Mitigation Strategies

* Secure Sudoers: Never grant sudo access to binaries that have "shell escape" capabilities (check GTFOBins).
* Code Sanitization: Remove all developer comments and credentials from production source code.
* Disable File Editing: Disable the WordPress Theme/Plugin editor by adding `define( 'DISALLOW_FILE_EDIT', true );` to `wp-config.php`.
{% endstep %}
{% endstepper %}

***
