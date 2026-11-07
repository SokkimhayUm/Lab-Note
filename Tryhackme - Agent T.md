
> **Lab:** TryHackMe — Agent T  
> **Link**: https://tryhackme.com/room/agentt
> **Local host:** `192.168.133.108`  
> **Target:** 10.49.169.164
> **Scope:** Reconnaissance, http enumeration, PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution, shell access, root access, Linux enumeration.

## Table of Content
```table-of-contents
```
---
## 1. Port scanning and service enumeration

First, let start by scan for open port with rustscan.

```bash
rustscan -a 10.49.169.164 -b 1000 --timeout 1500 -- -n -Pn
```

![Pasted image 20261018172537](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018172537.png)

Looks like only port 80 is open—likely a straightforward web exploit. Let's get more details on this port with nmap:

```bash
sudo nmap -sV -sC -O -P 80 -T4 10.49.169.164
```

![Pasted image 20261018172551](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018172551.png)

Run another whatweb scan:

```bash
whatweb http://10.49.169.164
```

![Pasted image 20261018172659](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018172659.png)

## 2. Search for known vulnerabilities

Alright, both tools show PHP 8.1.0-dev. Let's search Exploit-DB using the its CLI to see what we can find:

```bash
searchsploit PHP 8.1.0-dev
```

![Pasted image 20261018172809](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018172809.png)

Found an RCE exploit (ID: 49933.py). Copying it to the current working directory for modification:

```bash
searchsploit -m 49933
```

![Pasted image 20261018172932](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018172932.png)

## 3. Run exploit

The exploit looks ready to go without any modifications. Let's run it directly:

```bash
python 49933.py
```

![Pasted image 20261018173104](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018173104.png)

Nice, we got root access! That felt a bit too easy. 
The flag isn't in our current directory, so let's hunt for it. Since TryHackMe flags usually start with the word "flag", let's use that as our starting point:
## 4. Flag hunting

```bash
find / -type f -name "flag*" 2>dev/null
```

![Pasted image 20261018173118](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018173118.png)

Found it! This lab turned out to be incredibly straightforward. The flag was sitting right in the root directory:

```bash
cat /flag.txt
```

<details>
<summary>Spoiler Warning!</summary>

flag{4127d0530abf16d6d23973e3df8dbecb}

</details>

## 5. Defensive Remediation & Mitigations

To secure the asset against the specific attack vectors exploited in this assessment, the following remediation steps are recommended:

- **Upgrade Vulnerable PHP:** Replace the compromised PHP 8.1.0-dev build with a trusted, patched stable release to eliminate the associated backdoor vulnerability.
- **Enforce Strict File Validation:** Implement a file type whitelist and validate file contents. Store uploads outside the web root and disable script execution within upload directories.
- **Apply Least Privilege:** Run the web server under a dedicated, unprivileged account and restrict filesystem permissions to limit the impact of potential remote code execution (RCE).