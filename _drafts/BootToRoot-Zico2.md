---
layout: post
title: Boot To Root Walkthrough - Zico2
---

I recently volunteered for an initiative my employer is running called Security Champions, people working in tech teams who have a good understanding of security issues and best practises to act as advocates for security throughout the development lifecycle. After our initial sessions, the AppSec team who organised the event asked if anyone had any ideas for demos or topics they'd like to talk about for the next session.

As part of the day's activities, we'd split into small teams to tackle different vulnerabilities in [DVWA](http://www.dvwa.co.uk/). My team picked Command Injection, but as we worked through the different levels of input filtering, I explained we might be able to turn command injection into a reverse shell, which surprised the rest of my team. This got me thinking about how I could demonstrate how several vulnerabilities could be used together to totally compromise a server, which lead to me working on the Zico2 VM on [VulnHub](https://www.vulnhub.com).

For this walkthrough, I have a Kali VM with a pair of NICs; one in NAT mode and another on an internal network with no host connection on the 192.168.56.1/24 subnet. I also have the Zico2 VM on the same internal network on the 192.168.56.1/24 subnet and I've disabled the default NAT NIC.

To quickly determine the DHCP lease for the VM, I ran Nmap with `nmap -sn 192.168.56.1/24` to perform a ping sweep of my testing subnet and correlated the hosts that responded to the pings with the MAC address of the NIC attached to the target VM, which had acquired a lease for 192.168.56.101.

~~~
~/b/zico2 # ❯❯❯ nmap -sn 192.168.56.1/24                              
                                                                      
Starting Nmap 7.60 ( https://nmap.org ) at 2017-12-17 16:13 GMT       
Nmap scan report for 192.168.56.101                                   
Host is up (0.0011s latency).                                         
MAC Address: 00:0C:29:77:4F:04 (VMware)                               
Nmap scan report for 192.168.56.254                                   
Host is up (0.00028s latency).                                        
MAC Address: 00:50:56:FB:C7:0B (VMware)                               
Nmap scan report for 192.168.56.102                                   
Host is up.                                                           
Nmap done: 256 IP addresses (3 hosts up) scanned in 16.98 seconds     
~~~

Next, to get an idea of what TCP ports were open and what services were running on them, I ran Nmap again, this time using `nmap -Pn -sS -p- -T4 -sV -oA basic_nmap 192.168.56.101`. That long string of options breaks down to:

* `-Pn` - Don't Ping scan first, assume all hosts are up
* `-sS` - Perform a TCP SYN scan
* `-p-` - Shorthand for scan all ports
* `-T4` - Use agressive timing
* `-sV` - Attempt to perform service and version fingerprinting
* `-oA basic_nmap` - Output scan report in the 3 main report formats with a filename prefix of 'basic_nmap'

~~~
~/b/zico2 # ❯❯❯ nmap -Pn -sS -p- -T4 -sV -oA basic_nmap 192.168.56.101                            
                                                                                                  
Starting Nmap 7.60 ( https://nmap.org ) at 2017-12-17 16:14 GMT                                   
Nmap scan report for 192.168.56.101                                                               
Host is up (0.00024s latency).                                                                    
Not shown: 65531 closed ports                                                                     
PORT      STATE SERVICE VERSION                                                                   
22/tcp    open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)             
80/tcp    open  http    Apache httpd 2.2.22 ((Ubuntu))                                            
111/tcp   open  rpcbind 2-4 (RPC #100000)                                                         
60847/tcp open  status  1 (RPC #100024)                                                           
MAC Address: 00:0C:29:77:4F:04 (VMware)                                                           
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel                                           
                                                                                                  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .    
Nmap done: 1 IP address (1 host up) scanned in 26.96 seconds                                      
~~~

I wasn't really sure what the RPCbind service was for, so I spent a little while investigating it. Turns out that RPCbind is common used for services like NFS, so I wanted to see if that was a viable attack vector. After a bit of googling for information, I found the Nmap script for enumerating RPC, 'rpcinfo'. I also found the rpcinfo from some googling which needed me to install the nfs-common package.

~~~
~/b/zico2 # ❯❯❯ nmap --script rpcinfo 192.168.56.101                     
                                                                         
Starting Nmap 7.60 ( https://nmap.org ) at 2017-12-17 16:24 GMT          
Nmap scan report for 192.168.56.101                                      
Host is up (0.0014s latency).                                            
Not shown: 997 closed ports                                              
PORT    STATE SERVICE                                                    
22/tcp  open  ssh                                                        
80/tcp  open  http                                                       
111/tcp open  rpcbind                                                    
| rpcinfo:                                                               
|   program version   port/proto  service                                
|   100000  2,3,4        111/tcp  rpcbind                                
|   100000  2,3,4        111/udp  rpcbind                                
|   100024  1          43238/udp  status                                 
|_  100024  1          60847/tcp  status                                 
MAC Address: 00:0C:29:77:4F:04 (VMware)                                  
                                                                         
Nmap done: 1 IP address (1 host up) scanned in 8.00 seconds              

~/b/zico2 # ❯❯❯ rpcinfo 192.168.56.101                                     
   program version netid     address                service    owner       
    100000    4    tcp6      ::.0.111               portmapper superuser   
    100000    3    tcp6      ::.0.111               portmapper superuser   
    100000    4    udp6      ::.0.111               portmapper superuser   
    100000    3    udp6      ::.0.111               portmapper superuser   
    100000    4    tcp       0.0.0.0.0.111          portmapper superuser   
    100000    3    tcp       0.0.0.0.0.111          portmapper superuser   
    100000    2    tcp       0.0.0.0.0.111          portmapper superuser   
    100000    4    udp       0.0.0.0.0.111          portmapper superuser   
    100000    3    udp       0.0.0.0.0.111          portmapper superuser   
    100000    2    udp       0.0.0.0.0.111          portmapper superuser   
    100000    4    local     /run/rpcbind.sock      portmapper superuser   
    100000    3    local     /run/rpcbind.sock      portmapper superuser   
    100024    1    udp       0.0.0.0.168.230        status     105         
    100024    1    tcp       0.0.0.0.237.175        status     105         
    100024    1    udp6      ::.236.248             status     105         
    100024    1    tcp6      ::.163.85              status     105         

~~~

At this point, I thought the RPCBind service might be a red herring, so I left my investigation of it there, but made a note to come back to it later if I got stuck.

