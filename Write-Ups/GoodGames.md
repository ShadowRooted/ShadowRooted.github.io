---
layout: default
title: GoodGames
---

# Enumeración

```bash
$ ping -c1 10.129.228.133
PING 10.129.228.133 (10.129.228.133) 56(84) bytes of data.
64 bytes from 10.129.228.133: icmp_seq=1 ttl=63 time=7.73 ms

--- 10.129.228.133 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 7.732/7.732/7.732/0.000 ms
```
$ nmap 10.129.228.133
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-25 09:04 CDT
Nmap scan report for 10.129.228.133
Host is up (0.0091s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http

$ nmap -sCV -p80 10.129.228.133
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-25 09:05 CDT
Nmap scan report for 10.129.228.133
Host is up (0.0076s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.51
|_http-title: GoodGames | Community and Store
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
Service Info: Host: goodgames.htb


$ echo '10.129.228.133 internal-administration.goodgames.htb' | sudo tee -a /etc/hosts


