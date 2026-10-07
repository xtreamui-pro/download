# Xtream UI Pro — download and installation

**Website: <https://download.xtream-ui.pro>**

This repository publishes Xtream UI Pro for self-hosting on Ubuntu 22.04 / 24.04:

- **[Releases](https://github.com/xtreamui-pro/download/releases)**: one package per CPU
  (`xtreampro-<version>-linux-amd64.tar.gz`, `…-linux-arm64.tar.gz`) and `SHA256SUMS`.
  `upgrade.sh --tag <version>` downloads from here.
- **[INSTALL.md](INSTALL.md)**: the full installation guide — every scenario (main server,
  edge node, domains and TLS, recorder) and every option of `install.sh`, `upgrade.sh`,
  `uninstall.sh` and `reset.sh`. The same text is `README.md` inside every package and the
  main part of the website.

Connectors for billing systems and shops (WHMCS, WooCommerce, Odoo, …) are published
separately at <https://plugins.xtream-ui.pro>.

## Quick start

```sh
TAG=v1.3.0; ARCH=amd64                     # see the website for the current version; arm64 on ARM
curl -fLO https://github.com/xtreamui-pro/download/releases/download/$TAG/xtreampro-$TAG-linux-$ARCH.tar.gz
curl -fLO https://github.com/xtreamui-pro/download/releases/download/$TAG/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
mkdir -p xtream && tar -xzf xtreampro-$TAG-linux-$ARCH.tar.gz -C xtream && cd xtream
sudo ./install.sh
```

Then follow *After the install (required)* in [INSTALL.md](INSTALL.md).

## Improving the guide

Found a step that is wrong or unclear? Open an issue or a pull request against
`INSTALL.md` — see [CONTRIBUTING.md](CONTRIBUTING.md). Problems with the software itself
can be reported as an issue here too.

## License

The software in the releases is proprietary: © Xtream UI Pro, all rights reserved. You may
download it and run it on your own servers to operate your own service; you may not resell,
redistribute or reverse engineer it. The installation guide is under CC BY 4.0. Full terms:
[LICENSE](LICENSE). The connectors at <https://plugins.xtream-ui.pro> are MIT-licensed.
