---
title: Pickle Rick
date: 2025-08-21
categories: [ctf, tryhackme]
tags: [linux, recon, web]     # TAG names should always be lowercase
---

Hi everyone.

This Capture the Flag is about a web server exploitation, focusing on reconnaissance and Linux knowledge, where the administrator let some credentials inside the source code, which allows access the web server and with help of some misconfigured Linux permissions, it’s able to escalate the privileges into root.
To avoid this kind of issue, the developers and the SA (System Administrator) should take care of not let credentials inside the source code and be careful to configure all the permissions inside the Linux server.

The first step is to reconnaissance the target. After doing it with nmap, is able to see two services running on the target server, such as ssh and http.

```
[john@linux ~]$ nmap -sV -A 10.10.139.242
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-15 20:06 -03
Nmap scan report for 10.10.139.242
Host is up (0.22s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 2d049338604bbc7e17dbf9a10595c753 (RSA)
|   256 47b5dcf82ea4e9ea42706f086dd86492 (ECDSA)
|_  256 a1706beb25dfedb0149d275afa2aca56 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Rick is sup4r cool
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.62 seconds
```

With that in mind, it’s valid to perform a recon over the webserver, using gobuster. Looking into the webpage source code, there is a comment with a username, R1ckRul3s.

```
[john@linux THM]$ gobuster dir -w /opt/wordlists/Discovery/Web-Content/common.txt -u 10.10.217.235
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.217.235
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/wordlists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.4
[+] Timeout:                 10s
===============================================================
2023/03/16 08:09:35 Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 292]
/.htpasswd            (Status: 403) [Size: 297]
/.htaccess            (Status: 403) [Size: 297]
/assets               (Status: 301) [Size: 315] [--> http://10.10.217.235/assets/]
/index.html           (Status: 200) [Size: 1062]
/robots.txt           (Status: 200) [Size: 17]
/server-status        (Status: 403) [Size: 301]
Progress: 4713 / 4714 (99.98%)
===============================================================
2023/03/16 08:11:28 Finished
===============================================================
```

After look into the “/robots.txt“, it’s possible to see a clearly intelligible word, “Wubbalubbadubdub”. Using a browser extension like Wappalyzer, helps to find more details of what the webserver is using. In this case, it’s using PHP, but it didn’t show anything related to PHP during the webserver discovery. Because of it, it’s time to rescan the target, but using a PHP wordlist.

```
[john@linux THM]$ gobuster dir -w /opt/wordlists/Discovery/Web-Content/Common-PHP-Filenames.txt -u 10.10.217.235 -t 10
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.217.235
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/wordlists/Discovery/Web-Content/Common-PHP-Filenames.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.4
[+] Timeout:                 10s
===============================================================
2023/03/16 08:15:49 Starting gobuster in directory enumeration mode
===============================================================
/login.php            (Status: 200) [Size: 882]
/portal.php           (Status: 302) [Size: 0] [--> /login.php]
Progress: 5154 / 5164 (99.81%)
===============================================================
2023/03/16 08:17:51 Finished
===============================================================
```

There is a login page, and using the username R1ckRul3s and the clearly intelligible word, Wubbalubbadubdub, as password, it’s able to login into the webserver.

Everything but the command panel is blocked. The first nmap scan made, showed that the server is running in a Ubuntu server. So after testing a couple of Linux commands inside this panel, it’s remarkable some points:

- There is two home directories.

```
drwxrwxrwx 2 root   root   4096 Feb 10  2019 rick
drwxr-xr-x 4 ubuntu ubuntu 4096 Feb 10  2019 ubuntu
```
- The user is the www-data

```
www-data
```

- The user can execute any command as root without password.

```
Matching Defaults entries for www-data on ip-10-10-217-235.eu-west-1.compute.internal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-10-217-235.eu-west-1.compute.internal:
    (ALL) NOPASSWD: ALL
```

- There is the “super secret pickle ingred” file.

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

As the cat command is blocked, the unique way to see the content of those files is to accessing them in the browser by accessing ```<target-ip>/Sup3rS3cretPickl3Ingred.txt```, for example. So, after accessing the pickle ingredient in the browser, it’s able to see the first ingredient that Rick needs. To get the second, which is inside the `/home/rick` directory, it’s necessary to copy it to the actual directory (/var/www/html), using `sudo cp /home/rick/second\\ ingredients ./secondflag.txt`. If the command be executed without the “sudo”, it will not copy the file ‘cause the permission and ownership. But, the www-data can use any command as root without password, that’s the reason of using sudo. To get the last flag and ingredient to help Rick change back to his human form, it’s necessary to look into the /root directory, which is located the last flag, copy it as the same way that the second has been copied and accessing it by the browser, since cat command is blocked.

That is it, Rick is in his human form again.