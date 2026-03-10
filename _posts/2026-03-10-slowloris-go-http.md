---
title: Slowloris Attack on Go HTTP Server
date: 2026-03-10
---

# Slowloris Attack on Go HTTP Server

## Introduction

Slowloris is a denial-of-service attack that keeps HTTP
connections open and gradually exhausts server resources.

This experiment demonstrates how a Go HTTP server behaves
under a Slowloris-style attack.

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
