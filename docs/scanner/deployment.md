# Scanner Deployment

## Docker

```sh
docker run -d \
  --name tlsentinel-scanner \
  -e TLSENTINEL_SERVER_URL=https://your-server:8080 \
  -e TLSENTINEL_SCANNER_TOKEN=scanner_your_token_here \
  ghcr.io/tlsentinel/tlsentinel-scanner:latest
```

## Binary

Download the appropriate binary for your platform from [GitHub Releases](https://github.com/tlsentinel/tlsentinel-scanner/releases).

| Platform | Binary |
|---|---|
| Linux x64 | `scanner-linux-amd64` |
| Linux ARM64 | `scanner-linux-arm64` |
| macOS ARM64 | `scanner-darwin-arm64` |
| Windows x64 | `scanner-windows-amd64.exe` |

## Windows Service

See [Windows Service](../scanner/windows-service.md) for running the scanner as a Windows Service.
