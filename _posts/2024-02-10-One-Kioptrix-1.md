---
title: One - Kioptrix 1 Writeup
date: 2024-02-10 22:00:00 +0530
categories: [Vulnhub, Days of Security]
tags: [vulnhub, metasploit, easy, exploit, vulnerability]
---

## Intro 

[Kioptrix 1](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/) is a boot2root style box, where the main objective is to gain root access on the machine in any way possible. For this machine, we don't get any flags to flaunt.

## Enumeration

To begin with, we will start a quick nmap scan to identify all the open ports and possibly all the services which are running on those open ports as well. I like to use the following tags and scan all the open ports on the machine. It might be time consuming but it is better to be thorough than missing something crucial.

```
nmap -sC -sV -sT -p- --open $IP

Nmap scan report for 192.168.0.108
Host is up (0.0021s latency).
Not shown: 65529 closed tcp ports (conn-refused)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey: 
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp   open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
111/tcp  open  rpcbind     2 (RPC #100000)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
139/tcp  open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp  open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: 400 Bad Request
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC4_128_EXPORT40_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|_    SSL2_DES_192_EDE3_CBC_WITH_MD5
|_ssl-date: 2024-02-10T09:00:53+00:00; +1h01m49s from scanner time.
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-09-26T09:32:06
|_Not valid after:  2010-09-26T09:32:06
1024/tcp open  status      1 (RPC #100024)

Host script results:
|_clock-skew: 1h01m48s
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.73 seconds
```

Here is a quick description of all the tags used - 
* -sC: Instrcut nmap to use all the default scripts
* -sV: Help identifying the services running on the open ports
* -sT: Will perform a TCP connect scan
* -p: Specify which port range to scan. Adding '-' at the end will scan all the ports
* --open: Only show open ports

Key things to notice here are that we have apache running on port 80 and 443 and we have samba up as well.

Webpage that apache is serving is nothing special. It is just an ancient apache default page with nothing hidden underneath. Directory and DNS scans net no result

![Apache](/assets/kioptrix1/apache.png)

Nothing was hidden in the source code either.

Since, we're dealing with a rather old version of apache it is not a bad idea to check for known vulnderabilities. Exploitdb is our friend :D

With a quick search we can find multiple buffer overflow exploits for the apache server.

![Apache DB](/assets/kioptrix1/apache_db.png)

Since the exploit is available, it is as easy as just downloading the exploit and running it. 

We'll move it from our local exploit-db. If you don't have it locally you can either get the exploit from the website or you can install it locally with the following command

```
sudo apt install exploit-db
```
## Exploitation Part 1

Let's grab our exploit 

```
cp /usr/share/exploitdb/exploits/unix/remote/47080.c . [You can find the exploit ID on the website]

gcc -o exploit 47080.c -lcrypto
```
We'll add -lcrypto as instructed in the exploit file. Complilation will generate warnings, but we can ignore them

If we try to run it, it will give us a very helpful "help" menu.

![apache_help](/assets/kioptrix1/apache_instructions.png)

The 0x6a and 0x6b seems to be the closest to the version which we are running, so lets try those with default connection value.

![bash_apache](/assets/kioptrix1/bash_apache.png)

And easy as that, we have a basic low-level shell! But we're still far from done!

We can notice, that during the process, it is trying to collect a file from the web but it is not able to get it. So we might be missing a part of the exploit.
I decided to manually download the file and serve it using an http server to the exploit.
To achieve that we need a small modification in the source code of the exploit and recomplie it afterwards.
I replaced the web address to my local machine's IP and port where I'll be starting my http server as shown below.

![code change](/assets/kioptrix1/update_exploit_apache.png)

![serve exploit](/assets/kioptrix1/root_apache.png)

Once we serve the file, we can see that we directly get root access! With this, we have successfully achieved root on the box.

## Exploitation Part 2

Remember we had a Samba share available with us? Let's take a look at it as well.
Enumeration can be done using enum4linux. It is a samba enumeration binary, built around tools like smbclient, rpclient, and nmblookup.

```
enum4linux $IP
```
![enum4linux](/assets/kioptrix1/enum4linux.png)

Going through the output it yielded nothing much of interest. 
But, we never found the version of samba. Neither with nmap nor with enum4linux. It is probably similar to Apache. A bit old and probably with publically available exploit. 

Let's try probing it more, this time with metasploit and see if we can find more info. There's a module in MSF console to enumerate samba version.

![samba version](/assets/kioptrix1/smb_version.png)

The version is 2.2.1a. Now we have two approaches, we can go ahead and try to exploit with metasploit or we can go ahead in the same way as we did with apache. Let do both!
Since we're already in msfconsole, let's do that one first.
Seaching for the samba version, we can see multiple exploit for different OS versions. Let's select the linux one

![msf samba](/assets/kioptrix1/samba_exp.png)

Set all the necessary values (RHOSTS)
```
set RHOSTS = 192.168.0.108
```
And select a TCP reverse shell as our payload and send it away! 

![msf exploit](/assets/kioptrix1/smb_shell_msf.png)

With this simple exploit, we got root shell. Onto the other method!
It is pretty much exactly the same thing as the apache one and the exploit is identical to the MSF one itself.
We'll not got in-depth with it. You can use the below screenshot as a reference. The exploit id is 22469

![manual exploit](/assets/kioptrix1/smb_man_shell.png)

With this, we're good to close this machine! If you found another way to root, feel free to share! 



