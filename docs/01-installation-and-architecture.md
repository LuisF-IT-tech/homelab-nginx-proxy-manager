
# Installation and Architecture

## Overview

This document covers the initial planning, architecture, and installation of Nginx Proxy Manager in the Proxmox homelab.

Nginx Proxy Manager was deployed as a dedicated LXC container on the OptiPlex Proxmox host. Its purpose is to provide centralized reverse proxying and trusted HTTPS access for internal web services.

## Objective

The goals of this deployment were to:

- Replace IP address and port-based access with readable hostnames
- Centralize HTTPS certificate management
- Provide trusted SSL certificates for internal services
- Keep management services private to the local network
- Create a scalable design for future homelab applications
- Separate reverse-proxy services from the applications being proxied

## Environment

### Network

| Setting | Value |
|---|---|
| Primary network | `192.168.50.0/24` |
| Default gateway | `192.168.50.1` |
| Internal DNS server | `192.168.50.10` |
| Internal domain structure | `home.hadesnetlab.com` |

### Proxmox Hosts

| Host | Purpose |
|---|---|
| Wyse Proxmox | Dedicated network-services host |
| OptiPlex Proxmox | Main virtualization and application server |

### Core Services

| Service | IP Address | Proxmox Host | Container ID |
|---|---:|---|---:|
| AdGuard Home | `192.168.50.10` | Wyse | `100` |
| Uptime Kuma | `192.168.50.11` | Wyse | `101` |
| Nginx Proxy Manager | `192.168.50.12` | OptiPlex | `102` |

## Architecture

The final traffic path is:

```
Client Device
     |
     | DNS request
     v
AdGuard Home
192.168.50.10
     |
     | Returns Nginx Proxy Manager IP
     v
Nginx Proxy Manager
192.168.50.12
     |
     | Matches requested hostname
     v
Internal Web Service
```

Example:

```
https://uptime.home.hadesnetlab.com
                 |
                 v
AdGuard resolves the name to 192.168.50.12
                 |
                 v
Nginx Proxy Manager receives the HTTPS request
                 |
                 v
Request is forwarded to http://192.168.50.11:3001
```

## Design Decisions

### Dedicated LXC Container

Nginx Proxy Manager was installed in its own LXC container instead of directly on the Proxmox host.

This provides:

- Service isolation
- Easier backups and snapshots
- Independent restart and maintenance
- Cleaner troubleshooting
- Reduced risk to the Proxmox host
- Easier migration to another Proxmox node

### Static IP Address

The Nginx Proxy Manager container was assigned:

`192.168.50.12`

A stable address is required because AdGuard Home sends all internal reverse-proxy hostnames to this IP.

Changing the Nginx Proxy Manager IP would break internal hostname resolution until the AdGuard DNS rewrite was updated.

### Internal-Only Deployment

The reverse proxy was designed primarily for internal access.

No router port forwarding was required for the initial deployment.

The following management interfaces remain private:

- Proxmox
- AdGuard Home
- Uptime Kuma
- Nginx Proxy Manager
- Router administration

SSL validation was completed through Cloudflare DNS rather than exposing an HTTP validation service to the internet.

### Separate DNS and Proxy Roles

AdGuard Home and Nginx Proxy Manager perform different functions.

AdGuard Home answers the question:

> Which IP address should this hostname use?

Nginx Proxy Manager answers the question:

> Which internal service should receive this web request?

The basic flow is:

```
Hostname
   |
   v
AdGuard DNS
   |
   v
Nginx Proxy Manager IP
   |
   v
Exact Proxy Host
   |
   v
Backend service IP and port
```

## Nginx Proxy Manager Installation

Nginx Proxy Manager was installed using the Proxmox Community Scripts helper.

The script was run from the shell of the OptiPlex Proxmox host.

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/nginxproxymanager.sh)"
```

The command was run on the Proxmox host, not inside another container.

## Container Configuration

The completed deployment uses:

| Setting | Value |
|---|---|
| Container ID | `102` |
| Hostname | `nginxproxymanager` |
| Proxmox host | OptiPlex |
| IP address | `192.168.50.12` |
| Network bridge | `vmbr0` |
| Gateway | `192.168.50.1` |
| DNS server | `192.168.50.10` |
| Container type | LXC |

The container uses the same primary LAN as the services it proxies.

## Initial Access

After installation, the Nginx Proxy Manager administration interface was opened at:

`http://192.168.50.12:81`

Port `81` is the administrative web interface.

The following ports are used by Nginx Proxy Manager:

| Port | Purpose |
|---:|---|
| `80` | Incoming HTTP traffic |
| `443` | Incoming HTTPS traffic |
| `81` | Nginx Proxy Manager administration |

Port `81` should not be forwarded through the router.


## Initial Verification

The following address confirmed that the Nginx Proxy Manager web interface was available:

`http://192.168.50.12:81`

The container could also be verified from the OptiPlex Proxmox host.

```
pct status 102
```

Expected status:

```
status: running
```

The container network could be tested with:

```
pct exec 102 -- ip address
```

External connectivity could be tested with:

```
pct exec 102 -- ping -c 4 1.1.1.1
```

Internal connectivity could be tested by pinging another homelab service:

```
pct exec 102 -- ping -c 4 192.168.50.10
```

## Service Verification

The main Nginx Proxy Manager service is named:

`npm`

The reverse-proxy web server is OpenResty.

Both services can be checked from the OptiPlex Proxmox shell:

```
pct exec 102 -- systemctl status npm openresty --no-pager -l
```

Expected result:

- `npm.service` is active
- `openresty.service` is active

The generated Nginx configuration can be validated with:

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

Expected result:

```
syntax is ok
test is successful
```

## Configuration Locations

Nginx Proxy Manager stores active Proxy Host configurations under:

```
/data/nginx/proxy_host/
```

Disabled or inactive host configurations may appear under:

```
/data/nginx/dead_host/
```

Per-host logs are stored under:

```
/data/logs/
```

These locations became important later when troubleshooting Proxy Hosts that appeared in the web interface but were not being loaded correctly by OpenResty.

## Reverse Proxy Operation

Each web service receives its own exact Proxy Host.

Example:

```
wyse.home.hadesnetlab.com
```

is forwarded to:

```
https://192.168.50.50:8006
```

Nginx Proxy Manager accepts the browser connection on standard HTTPS port `443`, so the backend port does not need to be entered in the browser.

The user opens:

`https://wyse.home.hadesnetlab.com`

Instead of:

`https://192.168.50.50:8006`

## Frontend and Backend Connections

A reverse proxy creates two separate connections.

### HTTPS Backend

```
Browser
   |
   | HTTPS
   v
Nginx Proxy Manager
   |
   | HTTPS
   v
Proxmox
```

### HTTP Backend

```
Browser
   |
   | HTTPS
   v
Nginx Proxy Manager
   |
   | HTTP
   v
Internal application
```

The browser connection can remain encrypted even when the backend service only supports HTTP.

## Backup Access

Direct IP addresses were retained as emergency management paths.

Examples:

| Service | Direct Address |
|---|---|
| Nginx Proxy Manager | `http://192.168.50.12:81` |
| AdGuard Home | `http://192.168.50.10` |
| Uptime Kuma | `http://192.168.50.11:3001` |
| Wyse Proxmox | `https://192.168.50.50:8006` |
| Router | `http://192.168.50.1` |

Direct access is useful when:

- AdGuard DNS is unavailable
- Nginx Proxy Manager is offline
- A Proxy Host is misconfigured
- The wildcard certificate has a problem
- Internal DNS resolution is not working

## Security Considerations

- The Nginx Proxy Manager admin port remains internal.
- No credentials are stored in this repository.
- No API tokens are stored in this repository.
- No private keys are stored in this repository.
- Administrative services are not intentionally exposed to the internet.
- Remote administration should use a VPN such as Tailscale.
- Proxmox hosts should be backed up before major networking changes.
- Snapshots should be taken before modifying critical proxy configurations.

## Outcome

Nginx Proxy Manager was successfully deployed as a dedicated LXC container on the OptiPlex Proxmox host.

The completed architecture provides:

- A centralized HTTPS entry point
- Service isolation through Proxmox LXC
- Clean internal hostnames
- Reusable reverse-proxy configuration
- Centralized certificate management
- Easier troubleshooting and expansion
- Emergency direct-IP access

The next stage of the project configured Cloudflare, the custom domain, DNS validation, the wildcard certificate, and AdGuard internal DNS.

## Next Document

[Cloudflare, DNS, and SSL](02-cloudflare-dns-and-ssl.md)
