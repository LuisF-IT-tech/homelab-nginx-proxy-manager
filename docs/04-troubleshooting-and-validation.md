
# Troubleshooting and Validation

## Overview

This document covers the problems encountered while deploying Nginx Proxy Manager and the troubleshooting process used to resolve them.

The main issues included:

- Incorrect wildcard Proxy Host configuration
- HTTP and HTTPS backend mismatches
- Missing generated Nginx configuration files
- `ERR_SSL_UNRECOGNIZED_NAME_ALERT`
- OpenResty using an older configuration
- `502 Bad Gateway`
- Incorrect router backend IP address
- Malformed HTTP headers from the router
- Router compatibility with HTTP/1.1

The troubleshooting process focused on identifying the exact failing layer before changing settings.

## Environment

| Component | Value |
|---|---|
| Internal network | `192.168.50.0/24` |
| Router | `192.168.50.1` |
| AdGuard Home | `192.168.50.10` |
| Uptime Kuma | `192.168.50.11` |
| Nginx Proxy Manager | `192.168.50.12` |
| Wyse Proxmox | `192.168.50.50` |
| NPM container ID | `102` |
| AdGuard container ID | `100` |
| Wildcard DNS record | `*.home.hadesnetlab.com → 192.168.50.12` |
| Wildcard certificate | `*.home.hadesnetlab.com` |

## Troubleshooting Method

The troubleshooting process followed this order:

1. Confirm DNS resolution.
2. Confirm the Nginx Proxy Manager container is reachable.
3. Confirm the Proxy Host exists.
4. Confirm Nginx generated a configuration file.
5. Confirm the backend IP and port.
6. Test the backend from the NPM container.
7. Validate the OpenResty configuration.
8. Review the per-host error log.
9. Make one configuration change at a time.
10. Retest the service.

This prevented unrelated settings from being changed before the actual cause was identified.

# Issue 1: Wildcard Proxy Host Conflict

## Issue

A wildcard Proxy Host was created to handle all internal hostnames.

The Proxy Host contained names similar to:

```
*.home.hadesnetlab.com
.home.hadesnetlab.com
```

The host forwarded all matching requests to the Wyse Proxmox server.

## Symptoms

- Multiple internal hostnames opened the same backend.
- Services were routed to Proxmox instead of their correct destinations.
- The browser displayed redirect errors.
- Exact Proxy Hosts appeared offline or were bypassed.

One browser error was:

```
ERR_TOO_MANY_REDIRECTS
```

## Troubleshooting

The wildcard entry was found under:

`Hosts → Proxy Hosts`

The wildcard Proxy Host was different from the wildcard DNS rewrite and wildcard certificate.

The three wildcard-related components were separated:

| Component | Wildcard Allowed |
|---|---|
| AdGuard DNS rewrite | Yes |
| SSL certificate | Yes |
| General Proxy Host | No |

## Root Cause

A wildcard Proxy Host sent every hostname to one backend service.

Each web application required a different IP address, port, and possibly a different backend protocol.

## Resolution

The wildcard Proxy Host was deleted.

The following wildcard configurations were retained:

```
*.home.hadesnetlab.com → 192.168.50.12
```

```
home.hadesnetlab.com
*.home.hadesnetlab.com
```

Exact Proxy Hosts were created instead:

- `wyse.home.hadesnetlab.com`
- `uptime.home.hadesnetlab.com`
- `adguard.home.hadesnetlab.com`
- `router.home.hadesnetlab.com`

## Outcome

Each hostname now routes to its correct backend service.

## Lesson Learned

Use:

- Wildcard DNS
- Wildcard certificate
- Exact Proxy Hosts

# Issue 2: Incorrect Backend Scheme

## Issue

A Proxy Host was configured with the wrong backend protocol.

For example, Proxmox port `8006` was initially configured with HTTP.

## Symptoms

- Proxy Host showed offline.
- Browser displayed redirect errors.
- Service failed even though the backend IP and port were correct.

## Troubleshooting

The direct service address was checked.

Wyse Proxmox opened directly through:

```
https://192.168.50.50:8006
```

Uptime Kuma opened directly through:

```
http://192.168.50.11:3001
```

## Root Cause

The Nginx Proxy Manager Scheme field did not match the protocol used by the backend service.

## Resolution

The scheme was matched to the direct service URL.

| Service | Scheme | Port |
|---|---|---:|
| Wyse Proxmox | HTTPS | `8006` |
| Uptime Kuma | HTTP | `3001` |
| AdGuard Home | HTTP | `80` |
| Router | HTTP | `80` |

## Outcome

The Proxy Hosts successfully connected to their backend services.

## Lesson Learned

The Scheme field controls the connection between Nginx Proxy Manager and the backend.

It does not control the browser connection.

# Issue 3: Unknown AdGuard Web Port

## Issue

The AdGuard Home web-interface port was not known.

Port `53` was visible because AdGuard provides DNS, but that port could not be used for the web Proxy Host.

## Symptoms

- AdGuard Proxy Host failed.
- The backend port was uncertain.
- Port `80`, `3000`, and other ports were considered.

## Troubleshooting

The first command was accidentally run from the Wyse Proxmox host instead of inside the AdGuard container.

The prompt showed:

```
root@wyse
```

The correct container was then checked from the Wyse host:

```
pct exec 100 -- ss -lntp | grep -i AdGuardHome
```

The output showed AdGuard listening on:

```
*:80
*:53
```

## Root Cause

The DNS service and management interface use different ports.

| Port | Purpose |
|---:|---|
| `53` | DNS queries |
| `80` | AdGuard Home web interface |

## Resolution

The AdGuard Proxy Host was configured with:

```
Scheme: http
Forward IP: 192.168.50.10
Forward Port: 80
```

## Commands Used

```
pct exec 100 -- ss -lntp | grep -i AdGuardHome
```

## Outcome

The AdGuard web interface became reachable through the correct backend port.

Final address:

```
https://adguard.home.hadesnetlab.com
```

# Issue 4: SSL Unrecognized Name Alert

## Issue

The browser displayed:

```
ERR_SSL_UNRECOGNIZED_NAME_ALERT
```

This happened while opening:

```
https://adguard.home.hadesnetlab.com
```

## Symptoms

- DNS resolved correctly.
- The wildcard certificate existed.
- AdGuard was listening on HTTP port `80`.
- The browser reached Nginx Proxy Manager but the TLS connection failed.

## Troubleshooting

### Confirm DNS Resolution

From Windows PowerShell:

```
nslookup adguard.home.hadesnetlab.com
```

Expected result:

```
Address: 192.168.50.12
```

This confirmed that the hostname reached Nginx Proxy Manager.

### Search for the Proxy Host Configuration

From the OptiPlex Proxmox shell:

```
pct exec 102 -- sh -c 'grep -R -n "adguard.home.hadesnetlab.com" /data/nginx/proxy_host /data/nginx/dead_host 2>/dev/null'
```

An active host should appear under:

```
/data/nginx/proxy_host/
```

An inactive host may appear under:

```
/data/nginx/dead_host/
```

Initially, the command returned no result.

### Verify NPM Services

```
pct exec 102 -- systemctl status npm openresty --no-pager -l
```

Both services were active.

### Validate OpenResty

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

Expected result:

```
syntax is ok
test is successful
```

## Root Cause

The AdGuard Proxy Host appeared in the Nginx Proxy Manager interface, but NPM had not generated an active configuration file for it.

Because no active server block existed for the hostname, OpenResty could not match the requested TLS name.

## Resolution

The AdGuard Proxy Host was deleted and recreated without SSL first.

Initial settings:

```
Domain: adguard.home.hadesnetlab.com
Scheme: http
Forward IP: 192.168.50.10
Forward Port: 80
SSL Certificate: None
Force SSL: Off
```

After saving, the generated configuration was verified:

```
pct exec 102 -- sh -c 'grep -R -n "adguard.home.hadesnetlab.com" /data/nginx/proxy_host 2>/dev/null'
```

The result showed:

```
/data/nginx/proxy_host/6.conf
```

The wildcard certificate was then added back and Force SSL was enabled.

## Outcome

The exact hostname was loaded into the active Nginx configuration and HTTPS began working.

# Issue 5: OpenResty Did Not Load the Updated Configuration

## Issue

The AdGuard Proxy Host configuration file existed and contained the correct SSL settings, but the browser still displayed:

```
ERR_SSL_UNRECOGNIZED_NAME_ALERT
```

## Symptoms

The active Proxy Host file contained:

```
listen 443 ssl;
server_name adguard.home.hadesnetlab.com;
ssl_certificate /etc/letsencrypt/live/npm-1/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/npm-1/privkey.pem;
```

The configuration file was correct, but the running OpenResty process did not recognize the hostname.

## Troubleshooting

The SSL-related configuration was checked with:

```
pct exec 102 -- sh -c 'grep -nE "listen .*443|server_name|ssl_certificate|ssl_certificate_key" /data/nginx/proxy_host/6.conf'
```

The complete Nginx configuration was tested:

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

The test succeeded.

A reload was attempted:

```
pct exec 102 -- systemctl reload openresty
```

The service returned:

```
Job type reload is not applicable for unit openresty.service
```

## Root Cause

The OpenResty process was still using an older loaded configuration.

The configuration on disk was correct, but the running service had not reloaded it.

## Resolution

OpenResty was restarted:

```
pct exec 102 -- systemctl restart openresty
```

The service status was checked:

```
pct exec 102 -- systemctl is-active openresty
```

Expected result:

```
active
```

## Outcome

Restarting OpenResty forced it to reread the active Proxy Host configurations.

The AdGuard HTTPS hostname began working again.

## Lesson Learned

When the configuration file is correct and `nginx -t` succeeds, restart OpenResty before changing DNS, ports, certificates, or backend settings.

# Issue 6: Router Returned 502 Bad Gateway

## Issue

The router Proxy Host returned:

```
502 Bad Gateway
```

The requested address was:

```
https://router.home.hadesnetlab.com
```

## Symptoms

- DNS resolved to Nginx Proxy Manager.
- The wildcard certificate was assigned.
- Nginx Proxy Manager displayed a 502 page.
- The router could be accessed directly.

## Troubleshooting

### Test HTTP Connectivity

From the OptiPlex Proxmox shell:

```
pct exec 102 -- curl -I http://192.168.50.1
```

The router returned:

```
HTTP/1.1 200 OK
```

### Test HTTPS Connectivity

```
pct exec 102 -- curl -kI https://192.168.50.1
```

HTTPS also returned a successful response.

This confirmed:

- The router was online.
- The NPM container could reach it.
- No firewall blocked the connection.
- Routing between the container and router worked.

## Root Cause

The first router Proxy Host configuration contained the wrong backend IP address:

```
192.168.10.1
```

The correct router IP was:

```
192.168.50.1
```

## Resolution

The router Proxy Host was changed to:

```
Scheme: http
Forward IP: 192.168.50.1
Forward Port: 80
```

## Outcome

The request reached the correct router, but a second router-specific 502 problem remained.

# Issue 7: Router Sent Invalid HTTP Headers

## Issue

After correcting the router IP, the router still returned:

```
502 Bad Gateway
```

## Symptoms

The router worked when accessed directly.

A header-only request through the Proxy Host also returned:

```
HTTP/1.1 200 OK
```

However, loading the full router web page in the browser produced a 502.

## Troubleshooting

### Confirm the Proxy Host File

```
pct exec 102 -- sh -c 'grep -R -n "router.home.hadesnetlab.com" /data/nginx/proxy_host 2>/dev/null'
```

The result showed:

```
/data/nginx/proxy_host/7.conf
```

This confirmed that NPM generated and loaded the exact Proxy Host.

### Review the Per-Host Error Log

After reproducing the 502, the log was checked:

```
pct exec 102 -- sh -c 'tail -n 60 /data/logs/proxy-host-7_error.log'
```

The important error was:

```
upstream sent "Content-Length" and "Transfer-Encoding" headers at the same time while reading response header from upstream
```

The upstream destination was:

```
http://192.168.50.1:80/
```

## Root Cause

The router’s embedded web server returned both:

```
Content-Length
```

and:

```
Transfer-Encoding
```

in the same HTTP response.

`Content-Length` defines a fixed response size.

`Transfer-Encoding: chunked` sends the response in separate chunks.

The router incorrectly sent both response methods at the same time.

Nginx rejected the malformed HTTP/1.1 response and returned a 502.

## Resolution

The router Proxy Host was configured to communicate with the router using HTTP/1.0.

The following configuration was added under the Proxy Host’s custom Nginx configuration:

```
proxy_http_version 1.0;
proxy_set_header Connection "close";
```

WebSocket support was disabled because WebSockets require HTTP/1.1.

## Final Router Configuration

### Details

| Setting | Value |
|---|---|
| Domain | `router.home.hadesnetlab.com` |
| Scheme | `http` |
| Forward IP | `192.168.50.1` |
| Forward Port | `80` |
| Websockets Support | Off |
| Block Common Exploits | Off |

### SSL

| Setting | Value |
|---|---|
| Certificate | Wildcard certificate |
| Force SSL | On |
| HTTP/2 Support | On |
| HSTS | Off |

### Custom Configuration

```
proxy_http_version 1.0;
proxy_set_header Connection "close";
```

## Final Traffic Flow

```
Browser
   |
   | HTTPS and HTTP/2
   v
Nginx Proxy Manager
   |
   | HTTP/1.0
   v
Router
```

## Outcome

The router interface successfully loaded through:

```
https://router.home.hadesnetlab.com
```

without returning a 502 error.

# Error Reference

## 502 Bad Gateway

Usually means Nginx Proxy Manager received the browser request but could not successfully communicate with or process the backend.

Common causes:

- Incorrect backend IP
- Incorrect backend port
- Incorrect HTTP or HTTPS scheme
- Backend service offline
- Firewall blocking the connection
- Backend returning an invalid response
- Certificate validation problems between NPM and an HTTPS backend

## ERR_SSL_UNRECOGNIZED_NAME_ALERT

Usually means the browser reached OpenResty, but OpenResty did not have an active HTTPS server block matching the hostname.

Common causes:

- Proxy Host configuration file not generated
- Proxy Host disabled
- SSL certificate not assigned
- OpenResty has not loaded the latest configuration
- Incorrect hostname in the Proxy Host
- Wildcard certificate does not cover the hostname

## ERR_TOO_MANY_REDIRECTS

Usually means the browser is being redirected repeatedly.

Common causes:

- Wrong backend scheme
- HTTP backend configured as HTTPS
- HTTPS backend configured as HTTP
- Conflicting Force SSL behavior
- Wildcard Proxy Host sending traffic to the wrong service
- Backend application forcing a conflicting URL

## Connection Refused

Usually means the IP is reachable but nothing is listening on the specified port.

Check:

- Backend service status
- Correct port
- Firewall
- Application listener
- Container status

## Timeout

Usually means the connection cannot reach the backend or the backend is not responding.

Check:

- Routing
- VLAN rules
- Firewall rules
- Device availability
- Incorrect IP
- Application failure

# Validation Commands

## DNS Resolution

Run from Windows PowerShell:

```
nslookup adguard.home.hadesnetlab.com
```

Expected result:

```
Address: 192.168.50.12
```

## Clear Windows DNS Cache

```
ipconfig /flushdns
```

## Test TCP Connectivity

```
Test-NetConnection 192.168.50.12 -Port 443
```

## Test HTTP Response

```
curl.exe -I http://router.home.hadesnetlab.com
```

## Test HTTPS Response

```
curl.exe -I https://router.home.hadesnetlab.com
```

## Check Container Status

Run from the Proxmox host:

```
pct status 102
```

Expected result:

```
status: running
```

## Check NPM and OpenResty

```
pct exec 102 -- systemctl status npm openresty --no-pager -l
```

## Validate OpenResty Configuration

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

Expected result:

```
syntax is ok
test is successful
```

## Restart OpenResty

```
pct exec 102 -- systemctl restart openresty
```

## Confirm OpenResty Is Active

```
pct exec 102 -- systemctl is-active openresty
```

Expected result:

```
active
```

## Search Active Proxy Hosts

```
pct exec 102 -- grep -R -n "HOSTNAME" /data/nginx/proxy_host
```

Example:

```
pct exec 102 -- grep -R -n "adguard.home.hadesnetlab.com" /data/nginx/proxy_host
```

## Search Active and Disabled Hosts

```
pct exec 102 -- sh -c 'grep -R -n "HOSTNAME" /data/nginx/proxy_host /data/nginx/dead_host 2>/dev/null'
```

## Review a Proxy Host Error Log

```
pct exec 102 -- sh -c 'tail -n 60 /data/logs/proxy-host-ID_error.log'
```

Example:

```
pct exec 102 -- sh -c 'tail -n 60 /data/logs/proxy-host-7_error.log'
```

## Test a Backend from the NPM Container

```
pct exec 102 -- curl -I http://BACKEND-IP:PORT
```

Example:

```
pct exec 102 -- curl -I http://192.168.50.1
```

## Test a Self-Signed HTTPS Backend

```
pct exec 102 -- curl -kI https://BACKEND-IP:PORT
```

Example:

```
pct exec 102 -- curl -kI https://192.168.50.1
```

## Check Listening Ports Inside an LXC

```
pct exec CONTAINER-ID -- ss -lntp
```

Example:

```
pct exec 100 -- ss -lntp | grep -i AdGuardHome
```

# Configuration File Locations

## Active Proxy Hosts

```
/data/nginx/proxy_host/
```

## Disabled or Dead Hosts

```
/data/nginx/dead_host/
```

## Per-Host Logs

```
/data/logs/
```

## Let’s Encrypt Certificates

```
/etc/letsencrypt/live/
```

These paths are located inside the Nginx Proxy Manager container.

# Recommended Troubleshooting Order

When a new Proxy Host fails, use this order.

## Step 1: Test the Backend Directly

Open the original IP address and port.

Example:

```
http://192.168.50.10
```

If the direct address fails, fix the backend service before troubleshooting Nginx Proxy Manager.

## Step 2: Confirm DNS

```
nslookup service.home.hadesnetlab.com
```

Expected result:

```
192.168.50.12
```

## Step 3: Test the Backend from NPM

```
pct exec 102 -- curl -I http://BACKEND-IP:PORT
```

This verifies the NPM container can reach the service.

## Step 4: Confirm the Proxy Host File

```
pct exec 102 -- grep -R -n "service.home.hadesnetlab.com" /data/nginx/proxy_host
```

## Step 5: Validate OpenResty

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

## Step 6: Review the Error Log

```
pct exec 102 -- tail -n 60 /data/logs/proxy-host-ID_error.log
```

## Step 7: Restart OpenResty When Necessary

```
pct exec 102 -- systemctl restart openresty
```

## Step 8: Test in a Private Browser Window

This avoids cached redirects, failed TLS sessions, and old cookies.

# Final Validation

The following internal HTTPS addresses were tested successfully:

- `https://wyse.home.hadesnetlab.com`
- `https://uptime.home.hadesnetlab.com`
- `https://adguard.home.hadesnetlab.com`
- `https://router.home.hadesnetlab.com`

Each hostname:

1. Resolves to `192.168.50.12`.
2. Reaches Nginx Proxy Manager.
3. Matches an exact Proxy Host.
4. Uses the wildcard certificate.
5. Forwards to the correct internal backend.
6. Loads without a certificate warning.

# Final Outcome

The Nginx Proxy Manager deployment was successfully completed and validated.

The final configuration provides:

- Trusted internal HTTPS hostnames
- Wildcard DNS resolution
- Automated certificate renewal
- Exact backend routing
- Centralized reverse-proxy management
- Detailed per-host troubleshooting
- Direct-IP emergency access
- A repeatable process for adding future services

The completed project also provided practical experience with:

- DNS resolution
- TLS hostname matching
- Reverse proxies
- HTTP response headers
- HTTP protocol versions
- Proxmox LXC management
- Linux service management
- Nginx configuration validation
- Application and network-layer troubleshooting

## Return to Project README

[Homelab Nginx Proxy Manager](../README.md)
