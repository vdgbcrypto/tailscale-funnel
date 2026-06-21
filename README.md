# tailscale-funnel

Hermes Agent skill: **Tailscale Funnel** — expose local services, files, or directories to the public internet via Tailscale.

## What it does

This skill enables an AI agent to configure and manage Tailscale Funnel, which routes public internet traffic to local services through Tailscale's relay servers — no firewall changes, no port forwarding, end-to-end TLS.

## Usage

When a user wants to expose a local service, share a file publicly, or set up a webhook endpoint, this skill provides the complete workflow including:

- Initial Funnel setup and policy configuration
- Sharing web apps, files, directories, and TCP services
- Background/persistent mode with `--bg`
- PROXY protocol for preserving client IPs
- Troubleshooting common issues

## Skill file

See [SKILL.md](SKILL.md) for the full reference.

## Installation

Install as a Hermes Agent skill:

```bash
hermes skills install tailscale-funnel
```

Or manually copy `SKILL.md` to your skills directory.

## Requirements

- Tailscale >= v1.38.3
- MagicDNS enabled
- HTTPS certificates provisioned
- `funnel` node attribute in tailnet policy

## License

MIT
