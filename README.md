# remote-agents

Pre-built macOS binary releases for [`remote-coding`](https://github.com/vivaxy/remote-coding).

## Install

Download the latest tarball from [Releases](https://github.com/vivaxy/remote-agents/releases), then:

```bash
tar -xzf remote-coding-<version>-darwin-arm64.tar.gz
xattr -d com.apple.quarantine ./remote-coding
mv ./remote-coding /usr/local/bin/
```

The binary is ad-hoc signed; the `xattr` step is required once per download to clear macOS Gatekeeper quarantine.
