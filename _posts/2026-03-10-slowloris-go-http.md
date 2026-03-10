---
title: Simulating a Slowloris Attack on a Go HTTP Server
date: 2026-03-10
---

**Disclaimer**

The code shown in this article is intended for educational and research purposes only. It should only be executed in controlled environments such as local labs or test servers. Do not run these experiments against systems without explicit permission.

[demo](https://github.com/stevenliou2016/devops-go-app/tree/main "Title")


## Introduction

Slowloris is a classic application-layer denial-of-service attack that keeps
HTTP connections open by sending partial HTTP headers.

Instead of flooding the server with traffic, the attacker slowly sends headers
to keep connections alive and exhaust server resources such as file descriptors.

In this article, we will:

• build a minimal Go HTTP server  
• implement a Slowloris client in Go  
• simulate the attack  
• observe server behavior  

---

## What is a Slowloris Attack

normal request:
```
GET / HTTP/1.1
Host: example.com
```

Slowloris request：
```
GET / HTTP/1.1
Host: example.com
X-Test: slow
X-Test: slow
X-Test: slow
...
```
but never send
```
\r\n\r\n
```

The server waits for the request to finish while the attacker keeps the
connection alive.

---

## Attack Behavior
```
connections increase
↓
FD increase
↓
goroutines increase
↓
server resources consumed
```

---

## Lab Environment

The experiments in this article were conducted in a controlled Docker
environment to simulate resource constraints.

```
Environment:

OS: Ubuntu 24.04
Go: 1.22
Docker: 28.2.2
```

Run the server container:
```
docker run --rm \
  --name slow-test \
  -p 8083:8080 \
  --memory=256m \
  --cpus=1 \
  --pids-limit=112 \
  --ulimit nofile=112:112 \
  go-server
```

This configuration limits the container to 112 file descriptors, making it
easier to observe resource exhaustion during the experiment.

---

## Minimal Go HTTP Server

The server will take default timeout value(0) if timeout is not set. [A Timeout of zero means no timeout](https://go.dev/src/net/http/server.go#L3001 "Title").
```
server := &http.Server{
    Addr: ":" + port,
    Handler: loggingMiddleware(mux),
}
```
endpoint：
```
/health
/version
```
Get server health status
```
$ curl localhost:8083/health
ok
```

---

## Adding Observability

import pprof
```
import _ "net/http/pprof"
```
Add pprof goroutine
```
mux.Handle("/debug/pprof/goroutine", pprof.Handler("goroutine"))
```

