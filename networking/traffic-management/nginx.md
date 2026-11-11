# nginx

## Summary

nginx (pronounced "engine x", first released by Igor Sysoev in 2004) is an open source web server, reverse proxy, load balancer and cache, also able to proxy raw TCP and UDP and mail protocols. Its performance comes from its architecture: a privileged master process reads configuration and binds ports, and a small number of single-threaded worker processes (typically one per CPU core) each handle thousands of connections with a non-blocking, event-driven loop (epoll on Linux) instead of a thread or process per connection. The same design makes configuration reloads graceful: new workers start with the new configuration while old ones finish their requests. nginx is one of the most widely used web servers and the basis of many Kubernetes ingress controllers.

## Prerequisites

- [Reverse proxy](reverse-proxy.md) and [Load balancing](load-balancing.md).
- [Processes](../../operating-systems/process-management/processes.md) and [Threads](../../operating-systems/process-management/threads.md) for the process model.

## Core concepts

### Why the architecture matters

The basis of any Unix application is the thread or process: a self-contained set of instructions the operating system schedules on a CPU core. (In Linux, threads and processes are nearly identical; they differ in how much memory they share.) Complex applications run several in parallel to use more cores and to do work concurrently, for example to handle many connections.

Processes and threads cost memory and OS resources and require context switches. Servers can manage many of them, but they struggle when memory or I/O load becomes excessive. The traditional design of one thread or process per connection is simple but does not scale to tens of thousands of mostly idle connections (the "C10K problem"). nginx was built to solve it.

*Analogy.* Restaurant waiters. The connection-per-thread model assigns one waiter to every table for the whole evening, even while guests read the menu; with 10,000 tables you need 10,000 waiters. nginx has a few very fast waiters, one per kitchen station, each circulating among hundreds of tables and acting only when a table is ready to order or a dish is ready. The analogy breaks for slow tasks: if a waiter must wait for something blocking (disk I/O on some systems), nginx hands it to helper threads so the waiter keeps circulating.

### Process model

The first figure shows the master process with its child processes: cache manager, cache loader and worker processes, which share memory for cache, session persistence, rate limits and session logs.

![nginx master process and child processes: cache manager (CM), cache loader (CL) and worker processes (W) that handle HTTP and other network traffic, with shared memory used for cache, session persistence, rate limits and session log. Graphic by F5 NGINX, kept for personal study.](../../_assets/networking/nginx-process-architecture.png)

- **Master process**: performs privileged operations (reading configuration, binding to ports such as 80 and 443, opening log files), then creates the child processes and supervises them.
- **Cache loader**: runs at startup to load metadata of the disk cache into memory, then exits; scheduled conservatively, so its resource use is low.
- **Cache manager**: runs periodically and prunes the disk caches to their configured sizes.
- **Worker processes**: do all the real work: handle network connections, read and write content on disk, and talk to upstream servers.

The recommended configuration, one worker per CPU core (`worker_processes auto;`), uses hardware most efficiently. While nginx is active, only the workers are busy. Each worker handles many connections in a non-blocking way, which reduces context switches. Workers are single-threaded, run independently, accept new connections and process them, and share data (cache metadata, session persistence, rate limit counters) through shared memory.

### The event loop and state machines

![Inside a worker: a non-blocking, event-driven processing engine with stream, HTTP and mail state machines, auxiliary threads for blocking operations such as disk I/O, connections to static content and disk cache, access and error logs, HTTP, mail and TCP proxying and FastCGI, uWSGI, SCGI and memcached gateways. Graphic by F5 NGINX, kept for personal study.](../../_assets/networking/nginx-worker-event-loop.png)

Workers wait for events on the listening sockets (coordinated with `accept_mutex` or kernel socket sharding with `reuseport`). A new incoming connection is an event; the connection is assigned to a *state machine*. The HTTP state machine is the most used, but nginx also has state machines for stream traffic (raw TCP and UDP) and for mail protocols (SMTP, IMAP and POP3). When an operation would block, the worker moves on to other connections and returns when the kernel reports the socket ready (`epoll`, `kqueue`).

The HTTP state machine processes a request through phases:

![HTTP request processing in nginx: read request headers (TLS decryption, keepalive, bandwidth shaping), identify the configuration block (request matching and rewriting), apply rate limits, perform authentication, generate content locally or proxy to an upstream service, apply response filters (gzip, TLS, headers, image filter, sub), log, with internal redirects and subrequests; upstream services over HTTP, HTTPS, FastCGI, uWSGI, SCGI and memcached with load balancing, session persistence, health checks, retries, concurrency control, keepalive management and response caching. Graphic by F5 NGINX, kept for personal study.](../../_assets/networking/nginx-state-machine.png)

```mermaid
flowchart LR
  read["Read request headers<br/>TLS, keepalive"] --> match["Find server and location<br/>rewrite"]
  match --> limit["Rate and connection limits"]
  limit --> auth["Access control and auth"]
  auth --> content["Generate content:<br/>static file or proxy_pass"]
  content --> filters["Response filters:<br/>gzip, headers, sub"]
  filters --> log["Access log"]
```

### Advantages of the design

nginx scales to hundreds of thousands of connections per worker. Each new connection costs one file descriptor and a small amount of memory in the worker, with very little extra overhead. Workers can be pinned to CPUs, and context switches are rare: they happen when there is no work to do. In the blocking, connection-per-process approach, every connection needs substantial extra resources and context switches are very frequent.

### Configuration updates without downtime

The small number of workers makes configuration updates, and even binary upgrades, efficient.

![Load new configuration with no downtime: after the configuration is updated on disk and the master receives SIGHUP (nginx -s reload), the master starts new worker processes with the new configuration; old workers complete existing transactions and exit gracefully, so nginx keeps running with no interruption. Graphic by F5 NGINX, kept for personal study.](../../_assets/networking/nginx-config-reload.png)

Reloading is simple, lightweight and reliable: `nginx -s reload` (or `systemctl reload nginx`) checks the configuration on disk and sends the master a `SIGHUP`. On SIGHUP the master:

1. reloads the configuration and forks a new set of workers, which immediately accept connections and process traffic with the new settings;
2. tells the old workers to exit gracefully: they stop accepting new connections, finish their current HTTP requests, close connections cleanly (no lingering keepalives) and exit once all are closed.

If the new configuration is invalid, the master keeps the old workers running and logs the error. A binary upgrade uses `SIGUSR2` to start a new master alongside the old one.

### Configuration structure

```nginx
user  nginx;
worker_processes  auto;              # one per CPU core
events { worker_connections 4096; }  # per worker

http {
    include       mime.types;
    sendfile      on;
    keepalive_timeout 65;

    upstream app { server 10.0.1.11:8080; server 10.0.1.12:8080; }

    server {                          # virtual host, selected by listen and server_name
        listen 80;
        server_name example.com;
        location /        { proxy_pass http://app; }
        location /static/ { root /var/www; expires 7d; }
    }
}

stream {                              # layer 4 proxying
    server { listen 5432; proxy_pass 10.0.2.5:5432; }
}
```

Contexts nest: `main`, `events`, `http` (with `upstream`, `server`, `location`), `stream` and `mail`. Requests pick a `server` by `listen` and `server_name`, then the most specific `location` (exact `=` match, then longest prefix, with regex locations checked in order).

### Editions and related projects

- **nginx open source**, now maintained by F5 (which acquired NGINX Inc. in 2019).
- **NGINX Plus**: the commercial edition with active health checks, live activity monitoring, dynamic reconfiguration API and support.
- **Ingress controllers**: the community `ingress-nginx` (Kubernetes project) and F5's NGINX Ingress Controller are different products. The community ingress-nginx controller was announced for retirement, with maintenance ending in March 2026, and users directed to Gateway API implementations (TODO: verify the final status).
- **Forks**: OpenResty (nginx with Lua), Angie, and freenginx (forked by a core developer in 2024).

## Worked example

Serve a site with HTTP to HTTPS redirect, a backend proxy and rate limiting, then reload safely. Untested; nginx 1.26 on Ubuntu 24.04, checked 2026-11.

```nginx
# /etc/nginx/conf.d/example.conf
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    http2 on;
    server_name example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location /api/ {
        limit_req zone=api burst=20 nodelay;      # 10 requests/s per client IP, bursts of 20
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
    location / { root /var/www/example; try_files $uri $uri/ /index.html; }
}
```

```bash
sudo nginx -t                 # syntax and file checks; never reload without it
sudo systemctl reload nginx   # SIGHUP: new workers, old ones drain
ps -o pid,ppid,etime,cmd -C nginx   # new worker PIDs with short elapsed time, same master PID
curl -sI http://example.com   # 301 to https
for i in $(seq 1 40); do curl -s -o /dev/null -w '%{http_code} ' https://example.com/api/ping; done
# 200 ... 200 503 503 ...    <- requests beyond the burst are rejected (limit_req_status sets 429 if preferred)
```

## Trade offs and when to use it

- **Use nginx** as a high-performance static file server, reverse proxy, TLS terminator and simple load balancer with a large ecosystem.
- **Consider HAProxy** for advanced load balancing and observability of upstreams, **Envoy** for dynamic, API-driven configuration and service meshes, **Caddy** for automatic HTTPS with minimal configuration, and **Traefik** for container-native discovery.
- Dynamic configuration (upstreams changing every minute) is limited in open source nginx; reloads or NGINX Plus APIs are needed.

## Common mistakes

- **Reloading without `nginx -t`.** Fix: always test first (CI can run it on configuration changes).
- **`proxy_pass` trailing slash confusion**: with `location /api/` and `proxy_pass http://app/;` the `/api/` prefix is replaced; without the slash it is kept. Fix: decide explicitly and test.
- **`worker_connections` too low** for proxying (each proxied request uses two connections) and too-low file descriptor limits. Fix: raise both (`worker_rlimit_nofile`, `LimitNOFILE`).
- **`if` inside `location` for complex logic** ("if is evil"). Fix: `map`, `try_files` or separate locations.

## Practice

1. Why does nginx need far fewer processes than a thread-per-connection server?
2. What happens to in-flight requests during `nginx -s reload`?
3. What do the cache loader and cache manager do?
4. Why is `worker_processes auto` the usual choice?

Answers:

1. Each single-threaded worker handles thousands of connections with a non-blocking event loop, switching only when sockets are ready.
2. Old workers stop accepting new connections, finish current requests and exit; new workers with the new configuration take new connections, so nothing is cut off.
3. The loader loads disk cache metadata at startup and exits; the manager periodically prunes the cache to its size limits.
4. It starts one worker per CPU core, which uses all cores without unnecessary context switching.

## Further reading

- [Inside NGINX: How We Designed for Performance and Scale](https://blog.nginx.org/blog/inside-nginx-how-we-designed-for-performance-scale) (Owen Garrett, NGINX, 2015): the architecture description and figures used here.
- [nginx documentation](https://nginx.org/en/docs/) (nginx project): directives and modules.
- [NGINX Admin Guide](https://docs.nginx.com/nginx/admin-guide/) (F5 NGINX).
- *The Architecture of Open Source Applications, Volume II*, chapter "nginx" (Andrew Alexeev, 2012).
