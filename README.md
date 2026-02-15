CTF Writeup: Agent Root

Target IP: 10.64.132.208
1. Information Gathering & Port Scanning

An initial port scan was conducted to identify open services and versions:
Port	State	Service	Version
21/tcp	open	

FTP
	

vsftpd 3.0.3 

22/tcp	open	SSH	

OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 

80/tcp	open	HTTP	

Apache httpd 2.4.29 

2. Web Enumeration

Directory discovery using gobuster with multiple wordlists yielded no results. However, a manual inspection of the page content revealed a clue: "from agent R".

User-Agent Fuzzing

I hypothesized that the server provides different content based on the User-Agent. Using curl -L to follow redirects and fuzzing the User-Agent header, I discovered that using "C" as the User-Agent revealed a secret message:

    "Attention chris, Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! From, Agent R" 

This confirmed the target username is chris.

3. Exploitation (FTP Brute Force)

Since the web message mentioned a weak password, I performed a brute-force attack against the FTP service using hydra and the rockyou.txt wordlist:
Bash

hydra -l chris -P /usr/share/wordlists/rockyou.txt 10.64.132.208 ftp

I successfully retrieved the password and logged in to download the available files:

Bash

mget *

4. Steganography & Data Extraction

Among the downloaded files was a note from Agent C to Agent J stating that the "alien-like photos" were fake and a login password was hidden inside a "fake picture".

Extracting the Hidden ZIP

Using binwalk, I identified and extracted a hidden ZIP file from one of the images:

Bash

binwalk -e <picture_name>

The ZIP was encrypted. I converted it to a hash format and cracked it using john the ripper:

Bash

zip2john 8702.zip > zip_name.hash
john zip_name.hash --wordlist=/usr/share/wordlists/rockyou.txt

Decoding the Passphrase

After unzipping the file with 7z, I found a .txt file containing a Base64 encoded string. Decoding this provided a passphrase.

Extracting SSH Credentials

Using the decoded passphrase, I used steghide to extract hidden data from another image:

Bash

steghide extract -sf <picture_name>.jpg

This revealed a new username and SSH password.

5. Initial Access & Privilege Escalation

I established an SSH connection and began local enumeration. Checking the sudo version and permissions, I identified a known bypass vulnerability.

Exploit:
Bash

sudo -u#-1 /bin/bash

This command granted root access immediately. The final flag was located at /root/root.txt.
