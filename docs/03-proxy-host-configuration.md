
# Proxy Host Configuration

## Overview

This document covers the Nginx Proxy Manager Proxy Hosts created for the homelab.

Each Proxy Host provides a clean HTTPS hostname and forwards requests to the correct internal service IP address and port.

Configured services include:

- Wyse Proxmox
- Uptime Kuma
- AdGuard Home
- TP-Link router

## Objective

The goals of this stage were to:

- Replace raw IP addresses and ports with readable hostnames
- Assign the wildcard SSL certificate to each service
- Configure the correct backend protocol and port
- Keep all administrative services internal
- Use exact Proxy Hosts for each backend
- Confirm each service worked through HTTPS

## Environment

| Service | Internal IP | Backend Port | Backend Scheme |
|---|---:|---:|---|
| Router | `192.168.50.1` | `80` | HTTP |
| AdGuard Home | `192.168.50.10` | `80` | HTTP |
| Uptime Kuma | `192.168.50.11` | `3001` | HTTP |
| Nginx Proxy Manager | `192.168.50.12` | `81` | HTTP |
| Wyse Proxmox | `192.168.50.50` | `8006` | HTTPS |

## Prerequisites

Before creating the Proxy Hosts, the following components were already configured:

### AdGuard Wildcard DNS Rewrite

```
*.home.hadesnetlab.com → 192.168.50.12
```

This sends all internal service hostnames to Nginx Proxy Manager.

### Wildcard SSL Certificate

```
home.hadesnetlab.com
*.home.hadesnetlab.com
```

The certificate was created through Let’s Encrypt using Cloudflare DNS validation.

## How Proxy Hosts Work

The DNS rewrite only sends the hostname to Nginx Proxy Manager.

Nginx Proxy Manager then examines the requested hostname and selects the matching Proxy Host.

Example:

```
https://uptime.home.hadesnetlab.com
                 |
                 v
AdGuard resolves the name to 192.168.50.12
                 |
                 v
Nginx Proxy Manager matches the exact hostname
                 |
                 v
http://192.168.50.11:3001
```

## Wildcard Design Rule

Wildcards are used in:

- AdGuard Home DNS
- The SSL certificate

Exact hostnames are used in:

- Nginx Proxy Manager Proxy Hosts

Correct design:

```
Wildcard DNS
     |
     v
Nginx Proxy Manager
     |
     v
Exact Proxy Host
     |
     v
Exact backend IP and port
```

A general wildcard Proxy Host should not be used because each service requires a different backend destination.

## Choosing HTTP or HTTPS

The Scheme field controls how Nginx Proxy Manager communicates with the backend service.

It does not control what the browser uses.

To determine the correct scheme, check the service’s direct address.

### HTTP Backend

Direct address:

```
http://192.168.50.11:3001
```

Proxy Host configuration:

```
Scheme: http
Forward Port: 3001
```

### HTTPS Backend

Direct address:

```
https://192.168.50.50:8006
```

Proxy Host configuration:

```
Scheme: https
Forward Port: 8006
```

A browser can use HTTPS even when the backend service uses HTTP.

Example:

```
Browser
   |
   | HTTPS
   v
Nginx Proxy Manager
   |
   | HTTP
   v
Uptime Kuma
```

## Standard Proxy Host Process

A new Proxy Host is created from:

`Hosts → Proxy Hosts → Add Proxy Host`

### Details Tab

Enter:

- Exact domain name
- Backend scheme
- Backend IP address
- Backend port
- Access list
- Optional security settings

### SSL Tab

Select:

`home.hadesnetlab.com, *.home.hadesnetlab.com`

Enable:

- Force SSL
- HTTP/2 Support

Leave disabled during initial deployment:

- HSTS Enabled
- HSTS Sub-domains

### Access List

The Proxy Hosts use:

`Publicly Accessible`

This means Nginx Proxy Manager does not add another login prompt.

It does not automatically expose the service to the internet.

The services remain internal because no public router port forwarding was configured.

# Wyse Proxmox

## Purpose

Provide trusted HTTPS access to the Wyse Proxmox management interface.

## Direct Address

```
https://192.168.50.50:8006
```

## Final Hostname

```
https://wyse.home.hadesnetlab.com
```

## Details Tab

| Setting | Value |
|---|---|
| Domain Name | `wyse.home.hadesnetlab.com` |
| Scheme | `https` |
| Forward Hostname/IP | `192.168.50.50` |
| Forward Port | `8006` |
| Access List | Publicly Accessible |
| Cache Assets | Off |
| Block Common Exploits | On |
| Websockets Support | On |

## SSL Tab

| Setting | Value |
|---|---|
| SSL Certificate | Wildcard certificate |
| Force SSL | On |
| HTTP/2 Support | On |
| HSTS Enabled | Off |
| HSTS Sub-domains | Off |

## Connection Flow

```
Browser
   |
   | HTTPS
   v
Nginx Proxy Manager
   |
   | HTTPS
   v
192.168.50.50:8006
```

WebSocket support was enabled because Proxmox uses interactive web-console functionality.

# Uptime Kuma

## Purpose

Provide trusted HTTPS access to the Uptime Kuma monitoring dashboard.

## Direct Address

```
http://192.168.50.11:3001
```

## Final Hostname

```
https://uptime.home.hadesnetlab.com
```

## Details Tab

| Setting | Value |
|---|---|
| Domain Name | `uptime.home.hadesnetlab.com` |
| Scheme | `http` |
| Forward Hostname/IP | `192.168.50.11` |
| Forward Port | `3001` |
| Access List | Publicly Accessible |
| Cache Assets | Off |
| Block Common Exploits | On |
| Websockets Support | On |

## SSL Tab

| Setting | Value |
|---|---|
| SSL Certificate | Wildcard certificate |
| Force SSL | On |
| HTTP/2 Support | On |
| HSTS Enabled | Off |
| HSTS Sub-domains | Off |

## Connection Flow

```
Browser
   |
   | HTTPS
   v
Nginx Proxy Manager
   |
   | HTTP
   v
192.168.50.11:3001
```

The frontend connection is encrypted even though the internal backend uses HTTP.

# AdGuard Home

## Purpose

Provide trusted HTTPS access to the AdGuard Home administration interface.

## Direct Address

```
http://192.168.50.10
```

## Final Hostname

```
https://adguard.home.hadesnetlab.com
```

## Finding the Web Interface Port

AdGuard Home initially appeared to use an unknown web port.

The configured listeners were checked from the Wyse Proxmox host:

```
pct exec 100 -- ss -lntp | grep -i AdGuardHome
```

The result showed:

```
*:80
*:53
```

The ports represented:

| Port | Purpose |
|---:|---|
| `80` | AdGuard Home web interface |
| `53` | DNS service |

Port `53` was not used in the Proxy Host because it handles DNS queries rather than the web interface.

## Details Tab

| Setting | Value |
|---|---|
| Domain Name | `adguard.home.hadesnetlab.com` |
| Scheme | `http` |
| Forward Hostname/IP | `192.168.50.10` |
| Forward Port | `80` |
| Access List | Publicly Accessible |
| Cache Assets | Off |
| Block Common Exploits | Optional |
| Websockets Support | Optional |

## SSL Tab

| Setting | Value |
|---|---|
| SSL Certificate | Wildcard certificate |
| Force SSL | On |
| HTTP/2 Support | On |
| HSTS Enabled | Off |
| HSTS Sub-domains | Off |

## Connection Flow

```
Browser
   |
   | HTTPS
   v
Nginx Proxy Manager
   |
   | HTTP
   v
192.168.50.10:80
```

## Backup Access

The direct address was retained:

```
http://192.168.50.10
```

This provides emergency access if internal DNS or Nginx Proxy Manager is unavailable.

# Router

## Purpose

Provide trusted HTTPS access to the router’s internal administration interface.

## Direct Address

```
http://192.168.50.1
```

## Final Hostname

```
https://router.home.hadesnetlab.com
```

## Backend Connectivity Test

Connectivity was tested from the Nginx Proxy Manager container:

```
pct exec 102 -- curl -I http://192.168.50.1
```

The router returned:

```
HTTP/1.1 200 OK
```

HTTPS connectivity was also tested:

```
pct exec 102 -- curl -kI https://192.168.50.1
```

HTTP port `80` was selected as the backend because it avoided the router’s self-signed certificate.

## Details Tab

| Setting | Value |
|---|---|
| Domain Name | `router.home.hadesnetlab.com` |
| Scheme | `http` |
| Forward Hostname/IP | `192.168.50.1` |
| Forward Port | `80` |
| Access List | Publicly Accessible |
| Cache Assets | Off |
| Block Common Exploits | Off |
| Websockets Support | Off |

## SSL Tab

| Setting | Value |
|---|---|
| SSL Certificate | Wildcard certificate |
| Force SSL | On |
| HTTP/2 Support | On |
| HSTS Enabled | Off |
| HSTS Sub-domains | Off |

## Router-Specific Custom Configuration

The router required the following custom Nginx configuration:

```
proxy_http_version 1.0;
proxy_set_header Connection "close";
```

This configuration was entered through the gear icon in the Proxy Host editor.

The final connection path is:

```
Browser
   |
   | HTTPS and HTTP/2
   v
Nginx Proxy Manager
   |
   | HTTP/1.0
   v
192.168.50.1:80
```

The reason for this workaround is documented in:

[ Troubleshooting and Validation](04-troubleshooting-and-validation.md)

# Validation

## DNS Resolution

From Windows PowerShell:

```
nslookup uptime.home.hadesnetlab.com
```

Expected result:

```
Address: 192.168.50.12
```

The same result should be returned for all proxied hostnames.

## HTTP Header Test

A proxied service can be tested with:

```
curl.exe -I http://router.home.hadesnetlab.com
```

A working HTTP Proxy Host returned:

```
HTTP/1.1 200 OK
Server: openresty
X-Served-By: router.home.hadesnetlab.com
```

## HTTPS Test

A service can be tested with:

```
curl.exe -I https://router.home.hadesnetlab.com
```

Expected result:

```
HTTP/1.1 200 OK
```

## Nginx Configuration Test

From the OptiPlex Proxmox host:

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

Expected result:

```
syntax is ok
test is successful
```

# Adding Future Services

The existing wildcard DNS rewrite and wildcard SSL certificate can be reused.

For each new web service:

1. Confirm the direct IP address.
2. Confirm the direct port.
3. Confirm whether it uses HTTP or HTTPS.
4. Create an exact Proxy Host.
5. Select the wildcard certificate.
6. Enable Force SSL.
7. Test DNS resolution.
8. Test the backend from the NPM container.
9. Test the final HTTPS hostname.
10. Keep the direct IP address documented as a backup.

Example:

```
newservice.home.hadesnetlab.com
```

Backend:

```
http://192.168.50.25:8080
```

No new AdGuard DNS rewrite is required because the wildcard record already covers the hostname.

# Services Not Suitable for a Standard Proxy Host

Standard Proxy Hosts are intended for HTTP and HTTPS web applications.

They should not normally be used for:

- DNS
- SSH
- SMB file sharing
- Remote Desktop
- Minecraft game traffic
- General TCP services
- General UDP services

Those services should use:

- Direct DNS records
- Their normal ports
- VPN access
- Nginx Proxy Manager Streams when appropriate
- Service-specific proxy solutions

# Security Considerations

- Do not expose Nginx Proxy Manager port `81` publicly.
- Do not expose Proxmox directly through router port forwarding.
- Keep router administration internal.
- Keep direct-IP backup addresses documented.
- Use strong application passwords.
- Use a VPN for remote management.
- Leave HSTS disabled until all hostnames are stable.
- Do not proxy services without confirming the backend protocol.
- Do not add wildcard Proxy Hosts that route every service to one backend.
- Review each application’s authentication before creating a Proxy Host.

# Final Configuration

| Hostname | Backend |
|---|---|
| `wyse.home.hadesnetlab.com` | `https://192.168.50.50:8006` |
| `uptime.home.hadesnetlab.com` | `http://192.168.50.11:3001` |
| `adguard.home.hadesnetlab.com` | `http://192.168.50.10:80` |
| `router.home.hadesnetlab.com` | `http://192.168.50.1:80` using HTTP/1.0 |

# Outcome

The homelab web services can now be accessed through trusted internal HTTPS hostnames.

The configuration provides:

- Clean service URLs
- Centralized SSL termination
- Reusable wildcard DNS
- Reusable wildcard certificates
- Backend protocol flexibility
- Exact service routing
- Emergency direct-IP access
- A repeatable process for adding future web applications

## Next Document

[Troubleshooting and Validation](04-troubleshooting-and-validation.md)
