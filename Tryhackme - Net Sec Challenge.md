> **Lab:** TryHackMe — Vulnerability Capstone  
> **Link**: https://tryhackme.com/room/netsecchallenge
> **Local host:** `192.168.133.108`  
> **Target:** 10.49.187.38
> **Scope:** TCP and UDP port scanning, service enumeration, HTTP header inspection, SMB/NetBIOS enumeration, FTP brute-force testing, credential discovery, FTP file retrieval, and Nmap NULL scan techniques.

```table-of-contents
```
---
## 1 TCP Port Scanning

Let's start by scanning the target for open TCP ports using RustScan.

```bash
rustscan -a 10.49.187.38 -b 1000 --timeout 1500 -- -n -Pn
```

![Pasted image 20261019104517](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019104517.png)

Since the target has quite a few open ports, let's save the discovered ports to a file. This will make it easier to reuse the port list in subsequent scans.

```bash
rustscan -a 10.49.187.38 -b 1000 --timeout 1500 -g | awk -F'->' '{print $2}' | tr -d '[]' > ports.txt
```

![Pasted image 20261019104910](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019104910.png)

![Pasted image 20261019104925](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019104925.png)

Now we can use Nmap to enumerate the discovered ports with service version detection, default scripts, and OS detection.

```bash
sudo nmap -sV -sC -O -p $(cat ports.txt) -T4 10.49.187.38 -oX service.xml
```

The results are interesting. We can even find two hidden flags in the HTTP and SSH service port.

![Screenshot 2026-10-19 105604 1](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Screenshot%202026-10-19%20105604%201.png)

![Pasted image 20261019105640](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019105640.png)

## 2 UDP Port Scanning

With so many TCP ports open, let's check UDP as well. 

```bash
sudo nmap -sU --top-ports 100 --max-retries 1 --min-rate 1000 -v 10.49.187.38
```

![Pasted image 20261019110259](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019110259.png)

Nice, we found an open UDP port! Let's enumerate it to learn more about the service.

```bash
sudo nmap -sU -sV -p 137 --script nbstat 10.49.187.38
```

The target running Samba so it likely a Linux Machine.

From the host name, it look like the default internal naming for Amazon Web Services (ASW) that assign to EC2 instance.

<20> flag would've been useful if we haven't already know port 139 and 445 are open from our TCP scan.

![Pasted image 20261019111135](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019111135.png)

## 3 FTP Enumeration and Credential Discovery

We already have two usernames from the challenge's social engineering clues: `eddie` and `quinn`.

Rather than spending more time discovering usernames, let's focus on the FTP service first. Question 7 hints that its authentication may be vulnerable to brute-force attacks.

We'll use Hydra to test the two known usernames against the rockyou password list.

```bash
hydra -L user.txt -P /home/kali/Desktop/rockyou.txt -s 10121 ftp://10.49.187.38
```

![Pasted image 20261019125713](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019125713.png)

> **Credentials**
> | Username | Password |
> | -------- | -------- |
> | eddie    | softball |
> | quinn    | cookie   |

With valid credentials in hand, let's connect to the FTP server and inspect the available files.

The `eddie` account doesn't appear to contain anything useful, but the `quinn` account has what we're looking for.

![Pasted image 20261019131458](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019131458.png)

We found the flag

This demonstrates why reusing known usernames can save time during enumeration. We already had the usernames, so the next step was to test the authentication mechanism rather than enumerate accounts from scratch.
## 4. Finding the Stealth Scan Flag

The final question is interesting because it asks us to perform an Nmap scan while minimizing the chance of detection.

My first attempt was to slow the scan down, spoof the MAC address, and add some padding to the packets.

```bash
sudo nmap -sS -T1 --spoof-mac apple --data-length 24 -p $(cat ports.txt) 10.49.187.38
```

However, even though the scan was configured to be slower and use a spoofed MAC address, I still didn't get the flag.

Time to revisit the Nmap options.

```bash
nmap -h
```

After reviewing the available scan types, I found the option the challenge was looking for: a **NULL scan**.

![Pasted image 20261019133813](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Pasted%20image%2020261019133813.png)

```bash
sudo nmap -sN 10.49.187.38
```

This scan uses TCP packets with no flags set. Unlike a conventional TCP SYN scan, a NULL scan relies on how the target responds to unusual TCP packets to infer port states.

![Screenshot 2026-10-19 133211](Image/Tryhackme%20-%20Net%20Sec%20Challenge/Screenshot%202026-10-19%20133211.png)

One important detail: NULL scans don't guarantee stealth. Firewalls, intrusion detection systems, and different TCP/IP implementations can affect the results. The goal here is to use the scan technique requested by the lab.
## 6. Defensive Remediation & Mitigations

> **Info**
> To reduce the attack surface and mitigate the reconnaissance and credential attacks demonstrated in this assessment, the following remediation steps are recommended:
> 
> - **Restrict Network Exposure:** Disable unnecessary services and apply firewall rules to limit access to exposed TCP and UDP ports, particularly nonstandard service ports.
> - **Strengthen FTP Authentication:** Replace plaintext FTP with SFTP or FTPS, enforce strong unique passwords, and implement rate limiting or temporary lockouts to mitigate brute-force attacks.
> - **Minimize Information Disclosure:** Remove sensitive information from HTTP headers and service banners to reduce unnecessary exposure of internal details.
> - **Implement Scan Detection:** Configure firewall logging and IDS/IPS rules to identify suspicious port scanning patterns, including NULL scans.
## 5. Challenge Answer

<details>
<summary>Spoiler Warning!</summary>


| Question                                                                                                                                                      | Answer                  |
| :------------------------------------------------------------------------------------------------------------------------------------------------------------ | :---------------------- |
| What is the highest port number that is open and less than 10,000?                                                                                            | 8081                    |
| There is an open port outside the common 1000 ports; it is above 10,100. What is it?                                                                          | 10121                   |
| How many TCP ports are open?                                                                                                                                  | 7                       |
| On port 80, what is the service version value?                                                                                                                | THM{MySpecialServer007} |
| What is the flag hidden in the SSH server header?                                                                                                             | THM{946219583339}       |
| We have an FTP server listening on a nonstandard port. What is the version of the FTP server?                                                                 | vsftpd 3.0.5            |
| We learned two usernames using social engineering: **eddie** and **quinn**. What is the flag hidden in one of these two account files and accessible via FTP? | THM{QUINN_IS_BACK007}   |
| Browsing to `http://10.49.187.38:8081` displays a small challenge that will give you a flag once you solve it. What is the flag?                              | THM{NMAP_CHAMPION001}   |

</details>
