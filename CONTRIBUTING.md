# Contributing

This repository holds the installation guide ([INSTALL.md](INSTALL.md)) and the releases of
Xtream UI Pro. The website <https://download.xtream-ui.pro> is generated from `INSTALL.md` on
every release.

## Reporting a problem

Open an issue with: the version (`cat /opt/xtream/VERSION`), the Ubuntu version
(`lsb_release -ds`), the command you ran, and its output. `sudo journalctl -u xtream-dashboard -n 100`
(or `xtream-api`, `xtream-worker`, `xtream-serve`) usually shows the error. **Remove passwords,
secrets from `/etc/xtream/xtream.env`, IP addresses you do not want public and customer data before
you paste anything.**

## Changing the guide

1. Fork, branch from `main`, edit `INSTALL.md` (GitHub-flavoured Markdown: headings, lists, tables,
   fenced code blocks; no raw HTML — it is not rendered on the website).
2. Keep commands copy-pasteable and test them on a fresh Ubuntu server when you can; say in the pull
   request what you tested.
3. Describe options exactly as the scripts accept them. When the guide and a script disagree, say so
   in an issue instead of guessing.

A merged change appears on the website and inside the packages with the next release.

## License of contributions

By opening a pull request you agree that your contribution to the guide is licensed under
CC BY 4.0, like the rest of the guide (see [LICENSE](LICENSE)). Do not submit text or images
you do not have the right to share.
