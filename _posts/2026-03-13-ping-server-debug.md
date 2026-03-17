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

The server was running Docker with the default bridge network enabled.

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

## First Hypothesis: Firewall

```
# Check firewall status
sudo ufw status
```

```
# Check netfilter rules
sudo iptables -L -n
```

No rules blocking ICMP → Firewall was not the cause.

## Packet Capture Investigation

```
# On target server
sudo tcpdump -n icmp
```

```
# On client
ping 10.10.50.10
```

The server received the ICMP echo request.  
10.20.17.25 → 10.10.50.10: ICMP echo request  
But, the client never received a reply.


## Checking Routing Decisions

```
# On target server
ip route get 10.20.17.25
# Output
# 10.20.17.25 dev docker0 src 10.20.0.1
```

The kernel believed that the destination was reachable via docker0.


## Inspecting the Routing Table

```
# On target server
ip route
# Output
default via 10.10.50.1 dev eth0
10.10.50.0/24 dev eth0
10.20.0.0/16 dev docker0
```

docker0 network overlaps with the client network.


## Root Cause

Docker bridge:
```
# docker0
10.20.0.1/16
```

client network:
```
10.20.16.0/22
```

Linux routing rule:
```
Longest Prefix Match
```

Result:
```
10.20.17.25 → docker0
ICMP request arrives via eth0
ICMP reply leaves via docker0
```


## The Fix

To avoid network collisions, the following settings are configured in "/etc/docker/daemon.json":
```
{
  "bip": "172.30.0.1/16",
  "default-address-pools": [
    {
      "base": "172.31.0.0/16",
      "size": 24
    }
  ]
}
```
After applying the configuration, restart the Docker service.
```
sudo systemctl restart docker
```
Check network
```
docker network ls
ip addr show docker0
```


## Why This Happens with Docker

```
# docker0 default network
# Default subnet: 172.17.0.0/16
# If the client environment also uses the 172.17.x.x network range,
# a subnet collision may occur.
```

## Lessons Learned

1. Always verify routing decisions with `ip route get`.
2. Packet capture can reveal asymmetric routing issues.
3. Docker default networks may overlap with corporate networks.
4. Longest prefix match determines routing behavior in Linux.


## Conclusion

A simple ping failure turned out to be a routing conflict caused by Docker's bridge network.  

Understanding how Linux routing works is essential when debugging containerized hosts.
