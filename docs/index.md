# TLSentinel

A self-hosted TLS certificate and infrastructure monitoring platform. Track certificate expiry, TLS hygiene, and cipher suite health across your infrastructure via lightweight distributed scanners.

## Components

| Component | Description |
|---|---|
| **Server** | REST API + Web UI, PostgreSQL backend |
| **Scanner** | Lightweight agent, polls the server for hosts to scan |

## Quick Links

- [Installation](getting-started/installation.md)
- [Configuration](getting-started/configuration.md)
- [Scanner Deployment](scanner/deployment.md)
- [API Reference](api.md)
