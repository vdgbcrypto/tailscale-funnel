---
name: tailscale-funnel
description: "Use when exposing a local service, file, or directory to the public internet via Tailscale Funnel. Covers setup, configuration, CLI flags, policy, troubleshooting, and examples. Triggers: tailscale funnel, expose local service, share file publicly, funnel URL, tailnet public access."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [tailscale, funnel, networking, expose, public-access, https, reverse-proxy]
    related_skills: []
---

# Tailscale Funnel

Expose local services, files, or directories to the public internet through Tailscale's relay servers — no firewall changes, no port forwarding, end-to-end TLS.

## Overview

Tailscale Funnel routes traffic from the broader internet to a local service on your tailnet. Traffic flows: **Internet → Funnel relay → encrypted Tailscale proxy → local service**. The relay never decrypts data.

**Key characteristics:**
- Generates a stable Funnel URL: `https://<device>.<tailnet>.ts.net`
- Only ports `443`, `8443`, `10000` are allowed
- TLS-only (HTTPS)
- End-to-end encryption; relay servers cannot decrypt traffic
- Subject to bandwidth limits (non-configurable)

**Tailscale Funnel vs. Tailscale Serve:**
- **Funnel** — public internet access (use this skill)
- **Serve** — tailnet-only access (private sharing within your Tailscale network)
- Same port cannot be used for both simultaneously; the most recent command wins

## Requirements

| Requirement | Detail |
|-------------|--------|
| Tailscale version | ≥ v1.38.3 |
| MagicDNS | Enabled for your tailnet |
| HTTPS certs | Valid certs for `<tailnet>.ts.net` |
| Policy | `funnel` node attribute in tailnet policy file |
| Allowed ports | `443`, `8443`, `10000` only |
| Platform | Any OS running Tailscale CLI (macOS file sharing needs open-source variant) |

## When to Use

- Expose a local web app or dev server to the public internet
- Share a file or directory publicly via HTTPS
- Host a website on a Raspberry Pi or home server
- Provide a stable webhook endpoint for GitHub, Stripe, etc.
- Demo a local project to someone outside your network

**Don't use for:**
- Tailnet-only sharing → use `tailscale serve` instead
- Non-TLS traffic → Funnel requires TLS
- Ports other than 443, 8443, 10000

## Quick Start

### Enable Funnel (once per tailnet)

```bash
tailscale funnel
```

This triggers the web UI to approve, creates certificates, and adds the `funnel` node attribute to your tailnet policy.

### Share a Local Web App

```bash
# Share a service on port 3000
tailscale funnel 3000
```

Output:
```
Available on the internet:
https://amelie-workstation.pango-lan.ts.net

|-- / proxy http://127.0.0.1:3000

Press Ctrl+C to exit.
```

### Share a File or Directory

```bash
# Share a single file
tailscale funnel /home/alice/blog/index.html

# Share a directory (produces a listing page)
tailscale funnel /home/alice/blog/
```

> **macOS note:** File/directory sharing requires the open-source variant of Tailscale, not the App Store version.

### Stop Sharing

Press `Ctrl+C` in the running funnel process, or use `tailscale funnel --disable`.

## CLI Reference

### Command Syntax

```bash
tailscale funnel [flags] <target>
```

### Flags

| Flag | Description | Values |
|------|-------------|--------|
| `--bg` | Run in background (persists across reboots) | |
| `--https=<port>` | Expose HTTPS server | `443`, `8443`, `10000` |
| `--proxy-protocol=<ver>` | Enable PROXY protocol (preserves client IP) | `1` or `2` (v2 recommended) |
| `--set-path=<path>` | Append URL path to base Funnel URL | e.g., `/static` |
| `--tcp=<port>` | Raw TCP forwarder | `443`, `8443`, `10000` |
| `--tls-terminated-tcp=<port>` | TLS-terminated TCP forwarder | `443`, `8443`, `10000` |
| `--yes` | Skip interactive prompts | |

### Target Types

| Type | Example | Notes |
|------|---------|-------|
| Port number | `3000` | Reverse proxy to `http://127.0.0.1:<port>` |
| URL | `localhost:3000` | Same as port, explicit host |
| File path | `/home/alice/index.html` | Serve a single file |
| Directory path | `/home/alice/blog/` | Serve directory listing |
| Static text | `text:"Hello, world!"` | Debug/testing |
| HTTPS URL | `https://localhost:8443` | Backend with valid cert |
| Insecure HTTPS | `https+insecure://localhost:8443` | Skip cert verification |
| TCP | `tcp://localhost:22` | Raw TCP forwarding |

### Subcommands

```bash
# Check active funnel servers
tailscale funnel status
tailscale funnel status --json   # machine-readable

# Remove all funnel configuration
tailscale funnel reset
```

### Turning Off a Server

Append `off` to the exact command used to start it. All original flags must be repeated:

```bash
# Started with:
tailscale funnel --https=443 --set-path=/foo /home/alice/index.html

# Stop with:
tailscale funnel --https=443 --set-path=/foo /home/alice/index.html off
# or:
tailscale funnel --https=443 --set-path=/foo off
```

### Persistence

| Mode | Behavior after reboot |
|------|----------------------|
| `--bg` | Auto-resumes sharing |
| No `--bg` | Stops; must restart manually |

## Policy Configuration

Funnel requires the `funnel` node attribute in your tailnet policy file. Default policy (auto-added when enabling):

```json
"nodeAttrs": [
  {
    "target": ["autogroup:member"],
    "attr":   ["funnel"]
  }
]
```

This grants Funnel access to all tailnet members. Restrict by editing `target` to specific users or groups.

**Via Admin Console:** Access controls → Funnel → *Add Funnel to policy*

## Common Recipes

### Expose a Dev Server (background, persistent)

```bash
tailscale funnel --bg --https=443 localhost:3000
```

### Serve a Static Site with Custom Path

```bash
tailscale funnel --bg --https=8443 --set-path=/static /var/www/static
```

### TCP Forwarder with PROXY Protocol (preserve client IP)

```bash
tailscale funnel --bg --proxy-protocol=2 --tcp=10000 tcp://127.0.0.1:22
```

### Webhook Endpoint for GitHub/Stripe

```bash
# On the machine running the webhook receiver:
sudo tailscale funnel --bg 3000
# → https://my-server.tailnet.ts.net → http://127.0.0.1:3000
```

### Share a Folder Publicly

```bash
mkdir -p /tmp/public
echo "Hello" > /tmp/public/hello.txt
sudo tailscale funnel /tmp/public
# → https://my-server.tailnet.ts.net/hello.txt
```

### SSH Access via Funnel (TLS-terminated TCP)

```bash
tailscale funnel --bg --tls-terminated-tcp=8443 tcp://localhost:22
```

## Troubleshooting

| Issue | Check | Fix |
|-------|-------|-----|
| `funnel` node attribute missing | Policy file lacks `funnel` attr | Add via Admin UI → Access controls → Funnel, or edit JSON manually |
| HTTPS certs missing/invalid | MagicDNS & HTTPS not enabled | Run `tailscale funnel` to auto-provision certs |
| User can't create funnel | Not in permitted `target` list | Update `nodeAttrs` in policy (requires Owner/Admin/Network admin) |
| Funnel URL not resolving | DNS propagation delay | Wait up to 10 minutes |
| Let's Encrypt rate limit | Frequent cert renewals | Wait up to 34 hours before retrying |
| Port conflict | Same port used for `serve` and `funnel` | The most recent command wins; stop the other one |
| `sudo` required | Binding to privileged ports (443) | Use `sudo` or choose 8443/10000 |

## Verification Checklist

- [ ] Tailscale ≥ v1.38.3 installed
- [ ] MagicDNS enabled for tailnet
- [ ] HTTPS certificates provisioned (`tailscale funnel` without args triggers this)
- [ ] `funnel` node attribute present in tailnet policy
- [ ] Target port is 443, 8443, or 10000
- [ ] Funnel URL resolves (check with `curl -v https://<device>.<tailnet>.ts.net`)
- [ ] Service is accessible from outside the tailnet
