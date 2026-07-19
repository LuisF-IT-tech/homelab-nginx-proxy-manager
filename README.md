
# Homelab Nginx Proxy Manager

End-to-end deployment of Nginx Proxy Manager, Cloudflare DNS validation, wildcard SSL certificates, AdGuard Home internal DNS, and HTTPS reverse proxying in a Proxmox homelab.

## Project Overview

This project replaced IP address and port-based access with clean internal HTTPS hostnames.

Before:

`https://192.168.50.50:8006`

After:

`https://wyse.home.hadesnetlab.com`

The environment uses:

- AdGuard Home for internal DNS resolution
- Nginx Proxy Manager for reverse proxying
- Cloudflare for domain registration and DNS automation
- Let’s Encrypt for trusted SSL certificates
- Proxmox LXC containers for hosting network services

The services remain internal to the home network. Public router port forwarding was not required because SSL certificate validation was completed using the Cloudflare DNS-01 challenge.

## Project Goals

- Install Nginx Proxy Manager on Proxmox
- Assign Nginx Proxy Manager a static IP address
- Purchase and configure a custom domain
- Create a restricted Cloudflare DNS API token
- Request a wildcard Let’s Encrypt certificate
- Configure internal DNS through AdGuard Home
- Replace IP-based service access with internal hostnames
- Encrypt browser traffic with trusted HTTPS certificates
- Troubleshoot reverse-proxy, SSL, redirect, and backend errors
- Document the completed deployment for future maintenance

## Final Architecture

```text
Client Device
     |
     | DNS query
     v
AdGuard Home
192.168.50.10
     |
     | *.home.hadesnetlab.com
     | resolves to 192.168.50.12
     v
Nginx Proxy Manager
192.168.50.12
     |
     | Routes requests by hostname
     v
Internal Web Services
```

Example request flow:

```text
https://uptime.home.hadesnetlab.com
                 |
                 v
AdGuard DNS → 192.168.50.12
                 |
                 v
Nginx Proxy Manager
                 |
                 v
http://192.168.50.11:3001
```

## Environment

| Component | Address | Purpose |
|---|---|---|
| Router | `192.168.50.1` | Default gateway and network management |
| AdGuard Home | `192.168.50.10` | Internal DNS and filtering |
| Uptime Kuma | `192.168.50.11` | Network and service monitoring |
| Nginx Proxy Manager | `192.168.50.12` | Reverse proxy and SSL termination |
| Wyse Proxmox | `192.168.50.50` | Network-services Proxmox host |
| OptiPlex Proxmox | `192.168.50.60` | Main Proxmox server |
| Internal network | `192.168.50.0/24` | Primary homelab subnet |
| Domain | `hadesnetlab.com` | Cloudflare-managed domain |

### Proxmox Containers

| Container | ID | Host |
|---|---:|---|
| AdGuard Home | `100` | Wyse |
| Uptime Kuma | `101` | Wyse |
| Nginx Proxy Manager | `102` | OptiPlex |

## Internal DNS Design

AdGuard Home uses one wildcard DNS rewrite:

```text
*.home.hadesnetlab.com → 192.168.50.12
```

This sends every internal service hostname to Nginx Proxy Manager.

Examples:

- `wyse.home.hadesnetlab.com`
- `uptime.home.hadesnetlab.com`
- `adguard.home.hadesnetlab.com`
- `router.home.hadesnetlab.com`

Nginx Proxy Manager then uses an exact Proxy Host entry to determine the correct backend IP address and port.

### Key Design Rule

- Wildcard record in AdGuard DNS
- Wildcard entry in the SSL certificate
- Exact hostnames in Nginx Proxy Manager

A wildcard Proxy Host was not used because every internal service requires its own backend destination.

## SSL Certificate

A wildcard Let’s Encrypt certificate was created using Cloudflare DNS validation.

Certificate names:

```text
home.hadesnetlab.com
*.home.hadesnetlab.com
```

The wildcard certificate covers hostnames such as:

```text
wyse.home.hadesnetlab.com
uptime.home.hadesnetlab.com
adguard.home.hadesnetlab.com
router.home.hadesnetlab.com
```

Certificate validation uses a restricted Cloudflare API token with permission to edit DNS records only for `hadesnetlab.com`.

No API tokens, passwords, private keys, or recovery information are included in this repository.

## Configured Proxy Hosts

| Hostname | Backend Destination |
|---|---|
| `wyse.home.hadesnetlab.com` | `https://192.168.50.50:8006` |
| `uptime.home.hadesnetlab.com` | `http://192.168.50.11:3001` |
| `adguard.home.hadesnetlab.com` | `http://192.168.50.10:80` |
| `router.home.hadesnetlab.com` | `http://192.168.50.1:80` |

### Backend Protocol Selection

The Nginx Proxy Manager scheme must match how the service is accessed directly.

For example:

```text
http://192.168.50.11:3001
```

Uses:

```text
Scheme: http
Port: 3001
```

A service accessed directly through:

```text
https://192.168.50.50:8006
```

Uses:

```text
Scheme: https
Port: 8006
```

The browser can still use HTTPS even when the internal backend uses HTTP.

## Router Compatibility Workaround

The router’s embedded web server returned both of these headers in the same response:

```text
Content-Length
Transfer-Encoding
```

Nginx rejected the malformed upstream response and returned:

```text
502 Bad Gateway
```

The router Proxy Host required the following custom Nginx configuration:

```
proxy_http_version 1.0;
proxy_set_header Connection "close";
```

WebSocket support was disabled because the router workaround requires HTTP/1.0 for the upstream connection.

Final traffic flow:

```text
Browser → HTTPS/HTTP2 → Nginx Proxy Manager → HTTP/1.0 → Router
```

## Troubleshooting Completed

The deployment included troubleshooting for:

- `502 Bad Gateway`
- `ERR_TOO_MANY_REDIRECTS`
- `ERR_SSL_UNRECOGNIZED_NAME_ALERT`
- Incorrect backend IP addresses
- HTTP and HTTPS scheme mismatches
- Missing Nginx Proxy Host configuration files
- OpenResty running with an older configuration
- AdGuard web-port discovery
- Router response-header incompatibility
- Wildcard Proxy Host conflicts

The troubleshooting process relied on DNS tests, backend connectivity tests, generated Nginx configuration files, service status checks, and per-host error logs.

## Documentation

1. [Installation and Architecture](docs/01-installation-and-architecture.md)
2. [Cloudflare, DNS, and SSL](docs/02-cloudflare-dns-and-ssl.md)
3. [Proxy Host Configuration](docs/03-proxy-host-configuration.md)
4. [Troubleshooting and Validation](docs/04-troubleshooting-and-validation.md)

## Security Considerations

- Nginx Proxy Manager port `81` remains internal.
- Proxmox management interfaces are not publicly exposed.
- AdGuard Home administration is not publicly exposed.
- Router management is not publicly exposed.
- No router port forwarding was required for certificate validation.
- Cloudflare DNS validation uses a restricted API token.
- HSTS remains disabled until all proxy configurations are stable.
- Direct IP addresses are retained as emergency management paths.
- Remote administration should use a VPN such as Tailscale instead of direct public exposure.

The `Publicly Accessible` option inside Nginx Proxy Manager only means NPM does not add an additional authentication prompt. It does not automatically make a service accessible from the public internet.

## Validation

DNS resolution can be tested from Windows PowerShell:

```
nslookup adguard.home.hadesnetlab.com
```

Expected result:

```text
Address: 192.168.50.12
```

Backend connectivity can be tested from the OptiPlex Proxmox host:

```
pct exec 102 -- curl -I http://192.168.50.10
```

OpenResty configuration can be validated with:

```
pct exec 102 -- /usr/local/openresty/nginx/sbin/nginx -t
```

Expected result:

```text
syntax is ok
test is successful
```

## Outcome

The homelab now uses trusted internal HTTPS hostnames instead of raw IP addresses and port numbers.

The final environment provides:

- Centralized internal DNS
- Automated wildcard certificate renewal
- Trusted browser certificates
- Clean service URLs
- Centralized reverse-proxy management
- Easier service expansion
- Improved troubleshooting visibility
- A reusable design for future homelab applications

Future services can be added by creating one exact Proxy Host in Nginx Proxy Manager. The existing wildcard DNS rewrite and wildcard SSL certificate can be reused.
