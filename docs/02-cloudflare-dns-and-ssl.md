
# Cloudflare, DNS, and SSL

## Overview

This document covers the domain registration, Cloudflare DNS configuration, API-token creation, wildcard SSL certificate issuance, and AdGuard Home internal DNS configuration used for the homelab reverse proxy.

The final design provides trusted HTTPS certificates for internal services without exposing those services to the public internet.

## Objective

The goals of this stage were to:

- Register a custom domain for the homelab
- Use Cloudflare to manage the domain and DNS
- Create a restricted API token for certificate validation
- Request a trusted wildcard certificate through Let’s Encrypt
- Configure internal DNS through AdGuard Home
- Send internal service hostnames to Nginx Proxy Manager
- Avoid opening router ports for certificate validation
- Reuse one certificate for multiple internal services

## Environment

| Component | Value |
|---|---|
| Registered domain | `hadesnetlab.com` |
| Domain registrar | Cloudflare Registrar |
| Internal namespace | `home.hadesnetlab.com` |
| AdGuard Home IP | `192.168.50.10` |
| Nginx Proxy Manager IP | `192.168.50.12` |
| Nginx Proxy Manager container | `102` |
| Certificate authority | Let’s Encrypt |
| Validation method | DNS-01 |
| DNS provider | Cloudflare |

## Domain Registration

The domain purchased for the project was:

`hadesnetlab.com`

The domain was registered through Cloudflare Registrar for two years.

Automatic renewal was enabled to reduce the risk of accidentally losing the domain.

The domain registration included:

- WHOIS privacy
- DNSSEC support
- Cloudflare DNS management
- Automatic renewal options

No separate web hosting, SSL package, website builder, or paid email service was required.

## Internal Naming Structure

The following internal namespace was selected:

`home.hadesnetlab.com`

Services use one hostname level below that namespace.

Examples:

- `wyse.home.hadesnetlab.com`
- `uptime.home.hadesnetlab.com`
- `adguard.home.hadesnetlab.com`
- `router.home.hadesnetlab.com`
- `npm.home.hadesnetlab.com`

Using `home` keeps internal infrastructure separate from any future public website, portfolio, or email configuration using the main domain.

## Why a Publicly Registered Domain Was Used

A normal public certificate authority such as Let’s Encrypt cannot issue a publicly trusted certificate for a private name such as:

`wyse.homelab.arpa`

Using a registered domain allows browsers and operating systems to trust the certificates without installing a private certificate authority on every device.

The services still remain private because their internal DNS records point to private IP addresses.

## Split DNS Design

The environment uses split DNS.

Cloudflare manages the public DNS zone and proves ownership of the domain during certificate validation.

AdGuard Home manages the internal hostname resolution used by devices on the local network.

The traffic flow is:

```
Internal Client
      |
      | DNS query for a homelab hostname
      v
AdGuard Home
192.168.50.10
      |
      | Returns 192.168.50.12
      v
Nginx Proxy Manager
192.168.50.12
      |
      | Routes by exact hostname
      v
Internal Service
```

The services do not need public DNS records pointing to the home public IP address.

## Cloudflare API Token

Nginx Proxy Manager needs temporary permission to create DNS records during certificate validation.

A restricted Cloudflare API token was created instead of using the Global API Key.

### Token Name

`Nginx Proxy Manager - hadesnetlab.com`

### Token Permission

| Setting | Value |
|---|---|
| Permission group | `Zone` |
| Permission | `DNS` |
| Access | `Edit` |
| Zone resource | `hadesnetlab.com` only |
| Client IP restriction | None |
| Expiration | None |

The final permission summary was:

`hadesnetlab.com - DNS:Edit`

## What the Token Can Do

The token can:

- Create DNS records
- Modify DNS records
- Delete DNS records
- Perform those actions only inside the `hadesnetlab.com` zone

Nginx Proxy Manager primarily uses it to create and remove temporary TXT records for certificate validation.

## What the Token Cannot Do

The restricted token cannot:

- Access Cloudflare billing
- Change the Cloudflare account password
- Transfer or sell the domain
- Modify domain registration ownership
- Manage unrelated domains
- Access the internal network directly
- Sign in to Nginx Proxy Manager

## Token Security

The API token must be treated like a password.

The original token was accidentally exposed in a screenshot during setup. That token was immediately rolled in Cloudflare and replaced.

The exposed token was invalidated before continuing.

The replacement token was:

- Stored securely
- Added directly to Nginx Proxy Manager
- Never committed to GitHub
- Never included in documentation
- Never included in screenshots

## DNS-01 Certificate Validation

The certificate was requested through:

`Certificates → Add Certificate → Let’s Encrypt via DNS`

Cloudflare was selected as the DNS provider.

The DNS-01 validation process works as follows:

1. Nginx Proxy Manager requests a certificate from Let’s Encrypt.
2. Let’s Encrypt provides a DNS verification challenge.
3. Nginx Proxy Manager uses the Cloudflare API token.
4. A temporary TXT record is created under `_acme-challenge`.
5. Let’s Encrypt checks the TXT record.
6. Domain ownership is confirmed.
7. Let’s Encrypt issues the certificate.
8. The temporary TXT record is removed.

This validation method does not require port `80` or port `443` to be forwarded from the internet.

## Wildcard Certificate

The certificate includes both:

```
home.hadesnetlab.com
*.home.hadesnetlab.com
```

The first name covers the base internal namespace:

`home.hadesnetlab.com`

The wildcard covers one hostname level underneath it.

Examples covered by the wildcard:

- `wyse.home.hadesnetlab.com`
- `uptime.home.hadesnetlab.com`
- `adguard.home.hadesnetlab.com`
- `router.home.hadesnetlab.com`
- `npm.home.hadesnetlab.com`

The wildcard does not cover deeper names such as:

`test.wyse.home.hadesnetlab.com`

## Certificate Configuration

| Setting | Value |
|---|---|
| Certificate type | Let’s Encrypt via DNS |
| Domain 1 | `home.hadesnetlab.com` |
| Domain 2 | `*.home.hadesnetlab.com` |
| Key type | `ECDSA 256` |
| DNS provider | Cloudflare |
| Propagation time | `120` seconds |
| Validation method | DNS-01 |

## Cloudflare Credentials Format

The Cloudflare credentials field used this format:

```
# Cloudflare API token
dns_cloudflare_api_token=YOUR_API_TOKEN
```

The real token was entered directly into Nginx Proxy Manager.

The real value must never be added to this repository.

## Certificate Storage Warning

Nginx Proxy Manager warned that the DNS credentials would be stored in plaintext inside its database and configuration files.

This is necessary because Nginx Proxy Manager must reuse the API token when renewing the certificate.

For that reason:

- The token was restricted to one zone
- The token was given only DNS edit permissions
- The container must remain protected
- Backups containing credentials must be secured
- Nginx Proxy Manager port `81` must remain internal

## Certificate Renewal

Nginx Proxy Manager manages renewal automatically.

During renewal, it repeats the DNS-01 validation process using the saved Cloudflare token.

No manual TXT record creation should be required during normal renewal.

## AdGuard Home DNS Rewrite

AdGuard Home provides internal DNS resolution for the homelab.

The final wildcard rewrite is:

```
*.home.hadesnetlab.com → 192.168.50.12
```

The answer points to Nginx Proxy Manager, not directly to the backend service.

This allows all internal service hostnames to reach the reverse proxy.

## Why the Rewrite Points to Nginx Proxy Manager

The hostname does not directly identify the final backend IP address.

Instead, every web-service hostname first reaches Nginx Proxy Manager.

Nginx then uses the requested hostname to select the correct backend.

Example:

```
adguard.home.hadesnetlab.com
            |
            v
AdGuard DNS returns 192.168.50.12
            |
            v
Nginx Proxy Manager matches the hostname
            |
            v
http://192.168.50.10:80
```

## Wildcard DNS Benefits

Using one wildcard rewrite eliminates the need to create a separate AdGuard record for every new web service.

The wildcard automatically resolves names such as:

- `service1.home.hadesnetlab.com`
- `service2.home.hadesnetlab.com`
- `futureapp.home.hadesnetlab.com`

A matching Proxy Host must still be created inside Nginx Proxy Manager before the hostname can open a service.

## Important Wildcard Rule

Wildcards are used in:

- The AdGuard DNS rewrite
- The SSL certificate

Wildcards are not used as one general Proxy Host for every service.

The correct structure is:

```
Wildcard DNS Rewrite
        |
        v
Nginx Proxy Manager
        |
        v
Exact Proxy Host
        |
        v
Exact Backend Service
```

## Incorrect Wildcard Proxy Host

A wildcard Proxy Host was briefly created with names similar to:

```
*.home.hadesnetlab.com
.home.hadesnetlab.com
```

That Proxy Host forwarded every matching hostname to one Proxmox backend.

This caused incorrect routing and redirect problems because unrelated services were sent to the same destination.

The wildcard Proxy Host was deleted.

The wildcard DNS rewrite and wildcard certificate were kept.

## DNS Verification

From Windows PowerShell, DNS resolution was tested with:

```
nslookup adguard.home.hadesnetlab.com
```

Expected result:

```
Name:    adguard.home.hadesnetlab.com
Address: 192.168.50.12
```

This confirms that the hostname resolves to Nginx Proxy Manager.

## DNS Cache Clearing

After DNS changes, the Windows DNS cache can be cleared with:

```
ipconfig /flushdns
```

The hostname can then be checked again:

```
nslookup adguard.home.hadesnetlab.com
```

## Certificate Verification

The certificate can be checked in Nginx Proxy Manager under:

`Certificates`

The certificate should show:

- `home.hadesnetlab.com`
- `*.home.hadesnetlab.com`
- Let’s Encrypt as the issuer
- A valid expiration date

After assigning the certificate to a Proxy Host, the browser should show a valid HTTPS connection without certificate warnings.

## No Public Port Forwarding Required

The DNS-01 method does not require inbound public access to the homelab.

The following ports were not opened solely for certificate validation:

- TCP `80`
- TCP `443`
- TCP `81`

Nginx Proxy Manager port `81` remains an internal management interface.

## Publicly Accessible Setting

Nginx Proxy Manager Proxy Hosts were configured with the access option:

`Publicly Accessible`

Inside Nginx Proxy Manager, this means NPM does not add an additional access-list login prompt.

It does not automatically expose the service to the internet.

Actual public exposure would depend on:

- Public DNS records
- Router port forwarding
- Firewall rules
- Public IP configuration
- Cloudflare proxy settings

Those were not configured as part of this internal deployment.

## Security Considerations

- Never commit the Cloudflare API token to GitHub.
- Never include the token in screenshots.
- Never use the Global API Key when a restricted token will work.
- Restrict the token to one DNS zone.
- Keep Nginx Proxy Manager port `81` private.
- Do not publicly expose Proxmox or router administration.
- Keep direct-IP management addresses documented.
- Store administrator credentials in a password manager.
- Revoke or roll the API token if it is exposed.
- Protect backups containing the Nginx Proxy Manager database.

## Final DNS and Certificate Design

### Cloudflare

```
Domain: hadesnetlab.com
Validation: DNS-01
API permission: DNS Edit for hadesnetlab.com only
```

### Certificate

```
home.hadesnetlab.com
*.home.hadesnetlab.com
```

### AdGuard Home

```
*.home.hadesnetlab.com → 192.168.50.12
```

### Nginx Proxy Manager

Exact Proxy Hosts are created for each service.

Examples:

```
wyse.home.hadesnetlab.com
uptime.home.hadesnetlab.com
adguard.home.hadesnetlab.com
router.home.hadesnetlab.com
```

## Outcome

Cloudflare, Let’s Encrypt, AdGuard Home, and Nginx Proxy Manager were successfully integrated.

The final configuration provides:

- A custom internal domain structure
- Trusted browser certificates
- Automatic certificate renewal
- Internal wildcard DNS resolution
- No certificate-validation port forwarding
- Reusable DNS and SSL configuration
- A secure foundation for future web services

New internal web services can reuse the same wildcard DNS rewrite and wildcard certificate.

Only a new exact Proxy Host needs to be created for each backend application.

## Next Document

[Proxy Host Configuration](03-proxy-host-configuration.md)
