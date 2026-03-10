---
title: Slowloris Attack on Go HTTP Server
date: 2026-03-10
---

## Introduction

Slowloris is a classic application-layer denial-of-service attack that keeps
HTTP connections open by sending partial HTTP headers.

Instead of flooding the server with traffic, the attacker slowly sends headers
to keep connections alive and exhaust server resources such as file descriptors.

In this article, we will:

• Build a minimal Go HTTP server  
• Simulate a Slowloris attack using a custom Go client  
• Observe the server behavior using pprof and Linux tools  
• Analyze resource exhaustion (FD, connections, goroutines)  
• Implement mitigations  

---

## Architecture

Test environment:  
attacker  
|  
| slow HTTP headers  
|  
Go HTTP Server (Docker)  

Docker configuration:

```bash
docker run --rm \
  --name slow-test \
  -p 8083:8080 \
  --memory=256m \
  --cpus=1 \
  --pids-limit=112 \
  --ulimit nofile=112:112 \
  go-server
