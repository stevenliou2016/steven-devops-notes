---
title: Why Can I Ping One Server but Not Another?
description: A Network Debugging Case Study on a Linux Host
date: 2026-03-13
---

## Introduction

I encountered a strange networking issue while testing connectivity between two servers.  

Both servers were reachable on the same internal network, but only one of them responded to ping.  

Server A responded normally.  
Server B did not respond at all.  

At first glance, this looked like a typical firewall problem, but the investigation revealed something much more subtle.  

## Environment

All IP addresses in this article are from a lab environment. I will reproduce this issue via docker.

Network topology
![Network topology]({{ "/assets/images/ping-server-debug/network_topology.webp" | relative_url }})

server B was running Docker with the default bridge network enabled.

## Initial Observation

The client ping server A successfully.
```
ping -c 3 10.10.50.9
PING 10.10.50.9 (10.10.50.9) 56(84) bytes of data.
64 bytes from 10.10.50.9: icmp_seq=1 ttl=63 time=0.140 ms
64 bytes from 10.10.50.9: icmp_seq=2 ttl=63 time=0.074 ms
64 bytes from 10.10.50.9: icmp_seq=3 ttl=63 time=0.096 ms

--- 10.10.50.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2025ms
rtt min/avg/max/mdev = 0.074/0.103/0.140/0.027 ms
```

The client ping server B.
```
ping -c 3 10.10.50.10
PING 10.10.50.10 (10.10.50.10) 56(84) bytes of data.

--- 10.10.50.10 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2057ms
```
The client receives no response.

server A ping server B successfully.
```
ping -c 3 10.10.50.10
PING 10.10.50.10 (10.10.50.10): 56 data bytes
64 bytes from 10.10.50.10: seq=0 ttl=64 time=0.087 ms
64 bytes from 10.10.50.10: seq=1 ttl=64 time=0.072 ms
64 bytes from 10.10.50.10: seq=2 ttl=64 time=0.092 ms

--- 10.10.50.10 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.072/0.083/0.092 ms
```

server B ping server A successfully.
```
ping -c 3 10.10.50.9
PING 10.10.50.9 (10.10.50.9): 56 data bytes
64 bytes from 10.10.50.9: seq=0 ttl=64 time=0.118 ms
64 bytes from 10.10.50.9: seq=1 ttl=64 time=0.087 ms
64 bytes from 10.10.50.9: seq=2 ttl=64 time=0.085 ms

--- 10.10.50.9 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.085/0.096/0.118 ms
```

## Packet Capture Investigation

Perform packet capture on server B.
```
# On server B
tcpdump -i any icmp
```
The client ping server B.
```
# On the client
ping -c 3 10.10.50.10
PING 10.10.50.10 (10.10.50.10) 56(84) bytes of data.

--- 10.10.50.10 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2025ms
```
Packet cature on server B
```
# On server B
tcpdump -i any icmp
tcpdump: data link type LINUX_SLL2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on any, link-type LINUX_SLL2 (Linux cooked v2), snapshot length 262144 bytes
09:31:40.839536 eth0  In  IP 10.20.17.25 > aea644c28045: ICMP echo request, id 52, seq 1, length 64
09:31:41.864143 eth0  In  IP 10.20.17.25 > aea644c28045: ICMP echo request, id 52, seq 2, length 64
09:31:42.888159 eth0  In  IP 10.20.17.25 > aea644c28045: ICMP echo request, id 52, seq 3, length 64
09:31:43.912070 lo    In  IP aea644c28045 > aea644c28045: ICMP host 10.20.17.25 unreachable, length 92
09:31:43.912086 lo    In  IP aea644c28045 > aea644c28045: ICMP host 10.20.17.25 unreachable, length 92
09:31:43.912097 lo    In  IP aea644c28045 > aea644c28045: ICMP host 10.20.17.25 unreachable, length 92
```
Server B received the ICMP echo request.  
10.20.17.25 → 10.10.50.10: ICMP echo request  
But, the client never received a reply.


## Checking Routing Decisions

Check routing decisions
```
# On server B
ip route get 10.20.17.25
10.20.17.25 dev docker0 src 10.20.16.1 uid 0
```
Check network interfaces
```
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if270: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 8a:bb:f7:1a:09:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.50.10/24 brd 10.10.50.255 scope global eth0
       valid_lft forever preferred_lft forever
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 52:9f:3e:30:29:87 brd ff:ff:ff:ff:ff:ff
    inet 10.20.16.1/20 brd 10.20.31.255 scope global docker0
       valid_lft forever preferred_lft forever
```
The kernel believed that the destination was reachable via docker0.


## Root Cause

The /etc/docker/daemon.json configuration sets the docker0 IP to 10.20.16.1 and the subnet to 10.20.16.0/20.
```
{
  "bip": "10.20.16.1/20"
}
```
Client network overlaps with docker0 subnet.
```
The client network is 10.20.16.0/22 (range: 10.20.16.0–10.20.19.255).
The Docker network is 10.20.16.0/20 (range: 10.20.16.0–10.20.31.255).
→ Overlap
```
Linux routing behavior
```
Linux selects routes based on longest prefix match Connected routes (like docker0) take precedence
```
Result
```
Reply packets are routed to docker0 instead of the actual gateway.
```


## The Fix

To avoid network collisions, these two networks must not overlap.  
The /etc/docker/daemon.json was configured as follows:
```
{
  "bip": "10.20.20.1/22"
}
```
After applying the configuration, restart the Docker service.
```
sudo systemctl restart docker
```
Check docker0 network
```
# On server B
ip addr show docker0
8: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether ae:af:72:50:e1:93 brd ff:ff:ff:ff:ff:ff
    inet 10.20.20.1/22 brd 10.20.23.255 scope global docker0
       valid_lft forever preferred_lft forever
```
The client ping server B successfully.
```
ping -c 3 10.10.50.10
PING 10.10.50.10 (10.10.50.10) 56(84) bytes of data.
64 bytes from 10.10.50.10: icmp_seq=1 ttl=63 time=0.148 ms
64 bytes from 10.10.50.10: icmp_seq=2 ttl=63 time=0.088 ms
64 bytes from 10.10.50.10: icmp_seq=3 ttl=63 time=0.107 ms

--- 10.10.50.10 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2031ms
rtt min/avg/max/mdev = 0.088/0.114/0.148/0.025 ms
```


## Why This Happens with Docker

The default docker0 network is 172.17.0.0/16 (range: 172.17.0.0–172.17.255.255).
If the client environment also uses the 172.17.x.x network range, a subnet collision may occur.  
In this lab, the docker0 network is configured as 10.20.16.0/20 (range: 10.20.16.0–10.20.31.255), while the client network is 10.20.16.0/22 (range: 10.20.16.0–10.20.19.255) to simulate the overlap.


## Lessons Learned

1. Always verify routing decisions with `ip route get`.
2. Packet capture can reveal asymmetric routing issues.
3. Docker default networks may overlap with corporate networks.
4. Longest prefix match determines routing behavior in Linux.


## Conclusion

A simple ping failure turned out to be a routing conflict caused by Docker's bridge network.  

Understanding how Linux routing works is essential when debugging containerized hosts.
