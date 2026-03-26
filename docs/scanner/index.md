# Scanner

The TLSentinel scanner is a lightweight agent that polls the server for endpoints to scan and reports results back. Multiple scanners can be deployed across different network segments — each registers with the server using its own token.

Before deploying a scanner, create one in the server UI: **Settings → Scanners → New Scanner**. Copy the generated token before closing the dialog.

---

## Docker Compose (same host as server)

If you followed the [Installation guide](../getting-started/installation.md), add the scanner as an additional service in your existing `docker-compose.yml` and `.env`. See the **Adding a Scanner** section of that guide.

---

## Standalone Docker (remote host)

To monitor a separate network segment, run the scanner on a host with access to that network:

```sh
docker run -d \
  --name tlsentinel-scanner \
  --restart unless-stopped \
  -e TLSENTINEL_API_URL=https://your-server:8080 \
  -e TLSENTINEL_API_TOKEN=scanner_xxxxxxxxxxxxxxxxxxxx \
  ghcr.io/tlsentinel/tlsentinel-scanner:latest
```

Or with an env file:

```sh
# scanner.env
TLSENTINEL_API_URL=https://your-server:8080
TLSENTINEL_API_TOKEN=scanner_xxxxxxxxxxxxxxxxxxxx
```

```sh
docker run -d --name tlsentinel-scanner --restart unless-stopped \
  --env-file scanner.env \
  ghcr.io/tlsentinel/tlsentinel-scanner:latest
```

---

## Binary

Pre-built binaries are available on the [GitHub Releases](https://github.com/tlsentinel/tlsentinel-scanner/releases) page.

| Platform | Binary |
|---|---|
| Linux x64 | `scanner-linux-amd64` |
| Linux ARM64 | `scanner-linux-arm64` |
| macOS ARM64 | `scanner-darwin-arm64` |
| Windows x64 | `scanner-windows-amd64.exe` |

Set the required environment variables and run the binary directly. On Linux/macOS:

```sh
export TLSENTINEL_API_URL=https://your-server:8080
export TLSENTINEL_API_TOKEN=scanner_xxxxxxxxxxxxxxxxxxxx
./scanner-linux-amd64
```

---

## Windows Service

Running the scanner as a Windows Service is supported via [NSSM](https://nssm.cc) or the built-in `sc.exe`.

*Full Windows Service documentation coming soon.*

---

## Configuration Reference

| Variable | Description |
|---|---|
| `TLSENTINEL_API_URL` | **Required.** Base URL of the TLSentinel server |
| `TLSENTINEL_API_TOKEN` | **Required.** Scanner token generated in the server UI |

All other scanner behaviour (scan interval, concurrency, endpoint list) is controlled server-side and fetched automatically on connect.
