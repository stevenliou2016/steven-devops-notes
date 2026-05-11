---
title: Reproducing a Linux Routing Bug with Docker
description: How Overlapping Docker Networks Cause ICMP Reply Loss
date: 2026-05-11
---

# Reproducing a Linux Routing Bug with Docker

## 1. Introduction — From Debugging to Reproduction

In the previous article:

> *Why Can I Ping One Server but Not Another?*

we analyzed a real-world issue where:

- One server responds to ping
- Another does not
- Even though it receives the request

Understanding the issue is one thing.

Being able to **reproduce it in a controlled environment** is what allows us to truly validate and internalize the root cause.

In this article, we will:

- Recreate the issue using Docker
- Observe packet flow
- Validate Linux routing behavior
- Map the result back to real-world systems

---

## 2. Problem Recap

We start from the observed behavior:  

Client → Server A ✅ success  

Client → Server B ❌ fail

On Server B:

- ICMP request is received
- ICMP reply is generated
- But the reply never reaches the client

---

## 3. Lab Goal

Our goal is to reproduce the following conditions:

1. ICMP request arrives at Server B
2. Server B generates a reply
3. The reply is routed to the wrong interface

This is not a connectivity issue —  
it is a **routing decision problem**.

---

## 4. Lab Architecture

Network topology
![Network topology]({{ "/assets/images/ping-server-debug/network_topology.webp" | relative_url }})

### Design Principles

- Docker containers simulate hosts
- Docker networks simulate L2 segments
- A container acts as the router (gateway)
- An overlapping subnet is intentionally introduced

---

## 5. Quick Start

Clone and run:

```
git clone git@github.com:stevenliou2016/docker-network-overlap-lab.git
cd docker-network-overlap-lab
```

### option 1
docker compose up -d || docker-compose up -d

### option 2
./scripts/setup.sh

## 6. Environment Overview
| Component | IP                        | Description         |  
| --------- | ------------------------- | ------------------- |  
| Client    | 10.20.17.25               | Traffic source      |  
| Gateway   | 10.20.16.2 / 10.10.50.2   | Router              |  
| Server A  | 10.10.50.9                | Normal routing      |  
| Server B  | 10.10.50.10               | Overlapping network |  

## 7. Verification
### 7.1 Functional Test
```
docker exec client ping -c 2 10.10.50.9
```
Result: 
```
SUCCESS
```
---
```
docker exec client ping -c 2 10.10.50.10
```
Result: 
```
FAIL
```
### 7.2 Packet Observation
Install tcpdump:
```
docker exec serverB apk add tcpdump
```
Observe all interfaces:
```
tcpdump -i any icmp
```
You will see:
- ICMP request arrives
- ICMP reply is generated

But the reply does not return to the client.

### 7.3 Routing Decision (Critical)
```
docker exec serverB ip route get 10.20.17.25
```
Output:
```
10.20.17.25 dev docker0  src 10.20.16.1 
```
This reveals:
> The reply is routed to the Docker network interface, not the gateway

## 8. Root Cause
Server B routing table:
```
docker exec serverB ip route
default via 10.10.50.2 dev eth0 
10.10.50.0/24 dev eth0 scope link  src 10.10.50.10 
10.20.16.0/20 dev docker0 scope link  src 10.20.16.1
```
Client IP:
```
10.20.17.25 ∈ 10.20.16.0/20
```
Linux applies:
```
Longest Prefix Match
```
Comparison:
```
10.20.16.0/20   ← match
0.0.0.0/0       ← fallback
```
Result:
```
Packet is routed to docker network (incorrect path)
```

## 9. Why Server A Works
Server A does not have an overlapping subnet.

Its routing table only contains:
```
default via 10.10.50.2
```
So replies correctly go through the gateway.

## 10. Fix & Best Practices

### Why This Lab Uses a Fake `docker0`

In this lab, we intentionally create a fake `docker0` interface inside the container:

```bash
# create a fake docker0
docker exec serverB ip link add name docker0 type bridge
docker exec serverB ip addr add 10.20.16.1/20 dev docker0
docker exec serverB ip link set docker0 up
```

This approach allows us to:

- Reproduce the routing issue deterministically
- Simulate Docker bridge behavior
- Avoid dependency on Docker daemon internals
- Keep the lab lightweight and reproducible

Most importantly, the issue is not caused by Docker itself.

The real root cause is:

> An overlapping route with a more specific prefix.

The fake `docker0` simply reproduces the same routing condition.

---

### Lab Result

After creating the fake bridge, Server B routing table becomes:

```text
default via 10.10.50.2 dev eth0
10.10.50.0/24 dev eth0 scope link src 10.10.50.10
10.20.16.0/20 dev docker0 scope link src 10.20.16.1
```

For destination:

```text
10.20.17.25
```

Linux selects:

```text
10.20.16.0/20 dev docker0
```

instead of:

```text
default via 10.10.50.2
```

due to:

> Longest Prefix Match

As a result:

- ICMP reply is sent to `docker0`
- Packet never reaches the client
- Ping fails

---

### Real-World Fix (Production Environment)

In actual Docker environments, the recommended solution is:

### Avoid Overlapping Networks

Configure Docker address pools explicitly:

/etc/docker/daemon.json
```json
{
  "default-address-pools": [
    {
      "base": "172.31.0.0/16",
      "size": 24
    }
  ]
}
```

This prevents Docker from creating bridge networks that overlap with:

- corporate networks
- VPN ranges
- cloud VPC CIDR blocks
- Kubernetes pod networks

---

### Temporary Workaround

If overlap already exists, routing can be overridden manually:

```bash
ip route add 10.20.16.0/22 via 10.10.50.2
```

However, this is only a workaround.

The proper solution is still:

> Avoid overlapping CIDR ranges entirely.

## 11. From Lab to Production
This issue is not limited to Docker.

It can occur in:

- Kubernetes clusters
- Hybrid cloud networking
- VPN misconfiguration
- On-prem network overlaps

The underlying principle remains the same:
> Routing decisions are deterministic and based on prefix matching.

## 12. Key Takeaways
- Docker networks participate in Linux routing
- Overlapping CIDR blocks can silently break connectivity
- ICMP failure may be a routing issue, not a reachability issue
- Always verify routing decisions with:
```
ip route get <destination>
```

## 13. Appendix
Run Lab via Script
```
./scripts/setup.sh
```
Run Lab via Docker Compose
```
docker compose up -d || docker-compose up -d
```
Debug Commands
```
ip route
ip route get <ip>
tcpdump -i any icmp
```

## 14. Conclusion
In the previous article, we diagnosed the issue.

In this article, we:

- Reproduced it
- Observed it
- Verified the root cause

