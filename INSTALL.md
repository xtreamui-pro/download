# Xtream UI Pro — Installation guide (Ubuntu 22.04 / 24.04)

This guide is published at <https://download.xtream-ui.pro> and ships inside every
release package as `README.md`. It explains how to install, upgrade and remove
Xtream UI Pro on a bare Ubuntu server (VM or bare metal, systemd) with the
scripts in the package, and lists every option they take. Corrections are
welcome at <https://github.com/xtreamui-pro/download>.

**What the package holds**

| File | Purpose |
| --- | --- |
| `dashboard` | admin panel (port 8083) |
| `api` | subscriber API: Xtream Codes, MAG / Stalker portal, Enigma2, playlists, playback links, web player, Swagger (port 8082) |
| `worker` | background jobs: EPG, catalogue sync, backups, GeoIP, node installs over SSH |
| `seed` | creates / migrates the database schema and the first admin account |
| `serve` | media server: origin on the main server, cache + live fan-out on an edge node (port 8080) |
| `cluster` | edge node agent (enrol, heartbeat, commands); also inside the node package |
| `recording` | ffmpeg recorder with its own web UI (port 8084, optional) |
| `phpimport` | one-shot import of an existing PHP panel (categories, bouquets, groups, packages, users, lines) through its Admin API |
| `install.sh`, `upgrade.sh`, `uninstall.sh`, `reset.sh` | the scripts described below |
| `.env.example`, `examples/` | reference environment file and `serve` configurations (main / node) |
| `xtreampro-node-<version>-linux-<arch>.tar.gz` | the node package; `install.sh` copies it to `/opt/xtream/packages`, where the dashboard takes it from to install nodes over SSH |
| `deploy/monitoring/` | Prometheus / Grafana examples for the metrics endpoints |
| `VERSION`, `README.md`, `LICENSE` | the version stamp, this guide and the license of the software |

**What the package does not hold**

- The public website (`cmd/web`: landing, legal pages, help centre). It is not part of releases. Build it from source with `make build-web` when you want it; `install.sh` then creates the `xtream-web` unit because the binary sits next to the others.
- The billing / shop connectors (WHMCS, WooCommerce, Odoo, …). They are published on their own at <https://plugins.xtream-ui.pro> and can be downloaded from the dashboard (Connectors page).

The binaries are statically linked and packed with `upx --ultra-brute`: `file` reports `ELF … statically linked`, `upx -l <binary>` shows the ratio, and the first start of each process takes a few hundred milliseconds longer. Long-running services are not affected.

---

## 1. Get the package onto the server

Download **directly on the server** (uploading by hand often corrupts the file).
The current version and its packages are listed at <https://download.xtream-ui.pro>;
every version is kept at <https://github.com/xtreamui-pro/download/releases>.

```sh
TAG=v1.3.0; ARCH=amd64        # TAG = the version to install; ARCH=arm64 on an ARM machine (uname -m: x86_64 → amd64, aarch64 → arm64)
curl -fLO https://github.com/xtreamui-pro/download/releases/download/$TAG/xtreampro-$TAG-linux-$ARCH.tar.gz
curl -fLO https://github.com/xtreamui-pro/download/releases/download/$TAG/SHA256SUMS
```

Verify and extract:

```sh
sha256sum -c SHA256SUMS --ignore-missing          # must print ": OK"
mkdir -p xtream && tar -xzf xtreampro-$TAG-linux-$ARCH.tar.gz -C xtream
cd xtream && ls
#  api  cluster  dashboard  phpimport  recording  seed  serve  worker  install.sh  upgrade.sh  uninstall.sh  reset.sh
#  .env.example  examples/  deploy/  README.md  LICENSE  VERSION  xtreampro-node-<version>-linux-<arch>.tar.gz
```

Extract **on the server**. Files copied one by one through SFTP from Windows may lose the executable bit; the scripts still install (they `chmod 755` what they need), but run them with `bash ./install.sh` if the shell refuses to start them.

---

## 2. Scenario A — main server (panel + API + origin media)

The machine that runs the dashboard, the subscriber API, the worker and the origin media server. Run every command as root (`sudo`).

### A1. Simplest: reached by IP

```sh
sudo ./install.sh
```

A few minutes later:

| Component | Address |
| --- | --- |
| Dashboard | `http://<ip>:8083` |
| API + Swagger | `http://<ip>:8082/docs/` |
| Web player | `http://<ip>:8082/player/` |
| Media (serve) | `http://<ip>:8080` |

The database password, `JWT_SECRET`, `XTREAM_SECRET`, `CLUSTER_SYNC_TOKEN` and the admin password (`ADMIN_PASSWORD`) are generated randomly and stored in `/etc/xtream/xtream.env`. The admin password is printed **once** at the end of the run.

### A2. Domains, nginx in front, firewall

```sh
sudo ./install.sh --with-nginx \
  --dashboard-domain dash.example.com \
  --api-domain api.example.com \
  --media-domain media.example.com \
  --with-ufw
```

The script writes one nginx site for the three names and sets `DASHBOARD_URL=http://dash.example.com`, `API_PUBLIC_URL=http://api.example.com` (the host of the `get.php` links the dashboard builds for subscribers), `PLAYBACK_BASE_URL=http://media.example.com` and `CLUSTER_API_URL=http://api.example.com`. Add HTTPS afterwards:

```sh
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d dash.example.com -d api.example.com -d media.example.com
# re-run the installer with the same domain options: it keeps the certificates in the
# nginx site and switches the four public URLs in /etc/xtream/xtream.env to https://
sudo ./install.sh --with-nginx --dashboard-domain dash.example.com --api-domain api.example.com --media-domain media.example.com
```

A name behind Cloudflare (or any CDN that terminates TLS) is detected and `TRUSTED_PROXIES=private,cloudflare` is set so the API and the media server see the viewer's real address.

### A3. Edge nodes on other machines later

An enrolled node signs its cache-fill requests with its own key and the origin asks `cmd/api` for the list of valid nodes every minute, so **no** node IP has to be whitelisted. The whitelist only remains for legacy nodes that do not run the `cluster` agent:

```sh
sudo ./install.sh --with-nginx --dashboard-domain … --api-domain … --media-domain … \
  --node-whitelist 203.0.113.10,203.0.113.11
```

To add one after the install, edit the `node_whitelist` array in `/etc/xtream/serve.json` and `sudo systemctl restart xtream-serve`.

### A4. ffmpeg recorder

```sh
sudo ./install.sh --with-recording
```

The `xtream-recording` unit listens on port 8084; on its first start it prints its web account and API token to the journal: `journalctl -u xtream-recording | grep Generated`.

### After the install (required)

1. Sign in to the dashboard as `admin` with the password the installer printed (or `sudo grep ADMIN_PASSWORD /etc/xtream/xtream.env`) and **change it at once**. A later seed run (re-install, `upgrade.sh --seed`) never changes an existing account. The public showcase accounts (reseller, subreseller, user, demo) exist only when the install ran with `--with-demo`.
2. Dashboard → Settings → `cluster.origin_sync_token` = the value of `CLUSTER_SYNC_TOKEN` in `/etc/xtream/xtream.env` (`sudo grep CLUSTER_SYNC_TOKEN /etc/xtream/xtream.env`).
3. The sample subscriber `playerdemo` / `playerdemo` is for testing only; delete it before going public.
4. Optional: the worker downloads `GeoLite2-ASN.mmdb` and `GeoLite2-Country.mmdb` into `/var/lib/xtream/geoip/` within a minute or two (`journalctl -u xtream-worker | grep geoip`). For MaxMind's own files set `MAXMIND_ACCOUNT_ID` / `MAXMIND_LICENSE_KEY` in the env file, or copy the files there and `sudo systemctl restart xtream-api`.
5. Playback end to end: Dashboard → Tools → Playback check (pick a line and a channel).

---

## 3. Scenario B — edge node (media server close to the viewers)

A node runs `serve` in node mode (VOD cache + live fan-out) and the `cluster` agent only. No PostgreSQL or Redis.

### B1. What to collect from the main server

| Option | Where it comes from |
| --- | --- |
| `--main-url` | public URL of the origin media server, e.g. `https://media.example.com` |
| `--api-url` | public URL of `cmd/api`, e.g. `https://api.example.com` |
| `--public-url` | URL viewers use to reach this node, e.g. `https://edge1.example.com` (or `http://<node-ip>:8080`) |
| `--enroll-code` | Dashboard → Servers → add a server for the node → Cluster Nodes → **Create enrol code** (`XCE-…`, valid 30 minutes) |

### B2. Install and enrol in one command

```sh
sudo ./install.sh --role node \
  --main-url https://media.example.com \
  --api-url https://api.example.com \
  --public-url https://edge1.example.com \
  --enroll-code XCE-XXXX-XXXX-XXXX-XXXX \
  --with-ufw
```

A node **never receives** `XTREAM_SECRET`: it gets its own secret and private key when it enrols. `--api-url` must be `https://`; add `--allow-insecure` only on a test stack or a private network. A node installed from an old release: run `upgrade.sh` once, it removes `XTREAM_SECRET` from `/etc/xtream/xtream.env` and `secret` from `serve.json`.

### B2b. Install a node or a proxy from the dashboard over SSH

1. The main server needs the node package in `/opt/xtream/packages` (`xtreampro-node-<version>-linux-<arch>.tar.gz`). Every release package carries the one of its own architecture and `install.sh` copies it there; for another CPU run `make package-node GOARCH=arm64` from a source checkout and copy the file over.
2. Dashboard → Servers → **Add Server**: the IP (and **Server URL** when viewers reach the node by another name), and on the SSH tab an account that is root or has password-less `sudo` (for example `ubuntu` on a fresh Ubuntu machine). SSH passwords and keys are encrypted at rest; the worker checks the SSH access right after saving.
3. Dashboard → Servers → **Install Load Balancer**: pick the server → **Check SSH** (OS, CPU architecture, sudo, disk, clock drift, port 8080, reachability of `CLUSTER_API_URL`) → **Install**. Each step appears on the page as it runs. A main server installed by IP (plain-http API) has `CLUSTER_ALLOW_INSECURE=1` set for the worker; set it back to `0` once the API has an https name.
4. The same page offers **Upgrade**, **Restart services**, **Enroll again** and **Uninstall**; Servers → **Jobs** lists every job with its steps, commands and output (secrets filtered).
5. **Proxy** (Servers → Proxies → Add Proxy, pick the proxied server): installs nginx on the proxy machine with one site that forwards everything to the hidden server, checks `/healthz` through it and restarts the hidden node so it trusts the proxy's address.

The first SSH connection pins the machine's host key; after an OS re-install use Edit Server → SSH tab → **Reset host key**.

Check: `journalctl -u xtream-cluster-agent -f` must show `config applied`, and Dashboard → Cluster Nodes shows the node as **ok**.

### B3. Install first, enrol later

Leave `--enroll-code` out: the script installs, enables the units but does not start the agent, and prints the enrol command. When you have a code:

```sh
sudo -u xtream /opt/xtream/bin/cluster enroll --state /etc/xtream/cluster.json \
  --main https://api.example.com --code XCE-… --public-url https://edge1.example.com
sudo systemctl start xtream-cluster-agent
```

### When a node is revoked from the dashboard

The agent exits with code 3 and systemd does **not** restart it (`RestartPreventExitStatus=3`). To join again: create a new code, delete `/etc/xtream/cluster.json`, run the enrol command of B3 (or Dashboard → Install Load Balancer → **Enroll again**).

---

## 4. All options

### `install.sh`

| Option | Default | Meaning |
| --- | --- | --- |
| `--role main\|node` | `main` (or the role of an existing install) | Main server or edge node |
| `--bin-dir DIR` | `./build`, or `.` when the binaries sit next to the script | Where the binaries are taken from. Missing binaries on a machine with Go + pnpm trigger `make build` |
| `--with-nginx` | off | Install nginx, write `/etc/nginx/sites-available/xtream.conf`, disable the `default` site |
| `--dashboard-domain HOST`, `--api-domain HOST`, `--media-domain HOST` | empty | One name per service (with `--with-nginx`; any subset) |
| `--web-domain HOST` | empty | Root domain of the public site (`cmd/web`, unit `xtream-web`, port 8081); nginx serves `HOST` and `www.HOST`. Needs a `web` binary next to the others (`make build-web`): release packages do not carry one, so leave this out when installing a release — the script stops with a message otherwise |
| `--playback-base-url URL` | `http://<media-domain>` or `http://<ip>:8080` | URL viewers use for the origin media server, written to the env file |
| `--cluster-api-url URL` | `http://<api-domain>` or `http://<ip>:8082` | URL nodes use for `cmd/api`, written to the env file |
| `--node-whitelist LIST` | `127.0.0.1/32` | Comma-separated IPs / CIDRs of legacy nodes allowed on the origin's `/api/sync/*` |
| `--with-recording` | off | Enable the `xtream-recording` unit (port 8084) |
| `--with-ufw` | off | Install ufw and open the sshd port(s) in use (read from `sshd -T`, default 22) plus 80/443 (with nginx) or 8080–8083 (without); a node opens 8080 |
| `--with-fail2ban` | off | Install fail2ban with exactly two jails (`/etc/fail2ban/jail.d/xtream.local`): `sshd` (5 failures / 10 minutes) and `xtream-dashboard` (20 `401` answers from the dashboard in 10 minutes); 1-hour ban. A node gets the `sshd` jail only. `cmd/api` is never blocked |
| `--with-demo` | off | The seed also creates the sample catalogue and the public showcase accounts (`SEED_DEMO=1`). Not for a server open to the Internet |
| `--skip-seed` | seed runs | Do not run `seed` (binary upgrade without touching the schema) |
| `--no-tuning` | tuning on | Skip `sysctl.d/90-xtream.conf`, `limits.d/90-xtream.conf` and the PostgreSQL / Redis tuning |
| `--pg-version N` | `18` | PostgreSQL major version installed from apt.postgresql.org |
| `--main-url URL` | — | **Node**: origin media server to cache-fill from (required) |
| `--api-url URL` | — | **Node**: control plane `cmd/api` (required) |
| `--public-url URL` | — | **Node**: URL viewers use to reach this node (required to enrol) |
| `--enroll-code CODE` | — | **Node**: one-time code from Dashboard → Cluster Nodes; without it, enrol later |
| `--enroll-code-file FILE` | — | **Node**: read the code from FILE and delete the file (keeps the code out of the process list) |
| `--allow-insecure` | off | **Node**: enrol although `--api-url` is plain `http://` (test stacks, private networks) |
| `-h`, `--help` | | Print the usage |

`PLAYBACK_BASE_URL`, `CLUSTER_API_URL` and `node_whitelist` are written on the **first** install only; later changes go straight into the files. Secrets are never overwritten. The four public URLs of the env file follow the domain options of each run; a run without domain options leaves them alone.

### `upgrade.sh`

| Option | Meaning |
| --- | --- |
| `--tag vX.Y.Z` | Download this GitHub release (the asset for `uname -m`) and `SHA256SUMS`; default repository `xtreamui-pro/download` (public, no login), change with `--repo owner/repo` or the env `XTREAM_REPO`; a private repository needs `gh auth login` |
| `--file PKG` | Use a downloaded `.tar.gz` (verified when `SHA256SUMS` sits next to it) |
| `--dir DIR` | Use an already extracted package folder (default: the folder `upgrade.sh` runs from) |
| `--seed` | Also run `seed` (schema migration + missing defaults; existing accounts are never changed). Default: binaries only |
| `--no-tuning` | Pass `--no-tuning` to `install.sh` |
| `--rollback` | Restore the newest binaries backup from `/opt/xtream/backup` and restart. The database is **not** touched; the newest pre-upgrade dump and the `pg_restore` command are printed |
| `--require-backup` | Abort when the pre-upgrade `pg_dump` fails or `pg_dump` is missing (default: warn and go on) |
| `--yes`, `-y` | No confirmation prompt |
| `--` | Everything after it is passed to `install.sh` (for example `--with-recording`) |

### `uninstall.sh`

| Option | Meaning |
| --- | --- |
| (none) | Stop and remove the units, binaries, tuning files and the nginx site; `/etc/xtream` is archived to `/root/xtream-config-<timestamp>.tgz` and removed |
| `--purge-data` | Also delete `/var/lib/xtream` and `/var/cache/xtream` (media, recordings, GeoIP, node cache) and the `xtream` user |
| `--drop-db` | Also drop the database `xtreamprodb` and the role `xtream` |
| `--all` | `--purge-data --drop-db` |
| `--yes`, `-y` | No confirmation prompt |

### `reset.sh`

| Option | Meaning |
| --- | --- |
| (none) | Detects the mode: `/etc/xtream/xtream.env` → systemd install; a running `xtream-dashboard` / `xtream-api` container → docker; else development |
| `--mode install\|docker\|dev` | Force the mode |
| `--yes` | Skip the confirmation (typing the database name) |
| `--no-backup` | Skip the `pg_dump` taken before the reset |
| `--keep-redis` | Keep Redis database 0 instead of flushing it |

---

## 5. What the installer creates

| Path | Content |
| --- | --- |
| `/opt/xtream/bin/` | `dashboard api worker seed serve recording` (main) or `serve cluster` (node); `web` only when the binary was next to the others |
| `/opt/xtream/packages/` | the node package the dashboard installs nodes from over SSH |
| `/opt/xtream/VERSION` | version stamp (shown at the foot of the dashboard sidebar) |
| `/usr/local/bin/xtream-serve` | symlink to `serve`, for `xtream-serve sign "Film (2024)"` |
| `/etc/xtream/xtream.env` | environment variables and secrets, mode `root:xtream 640` |
| `/etc/xtream/serve.json` | `serve` configuration (main or node) |
| `/etc/xtream/cluster.json`, `node_runtime.json` | node: identity and runtime written by the agent |
| `/var/lib/xtream/media/` | VOD library, `enigma2/` (screenshots), `archive/` (live catch-up) |
| `/var/lib/xtream/geoip/` | `.mmdb` files |
| `/var/lib/xtream/backups/` | database dumps (worker schedule and pre-upgrade dumps), mode 700 |
| `/var/lib/xtream/recording/` | recorder `config.json` and recordings |
| `/var/cache/xtream/` | node VOD cache |
| `/etc/systemd/system/xtream-*.service` | `xtream-dashboard`, `xtream-api`, `xtream-worker`, `xtream-serve`, `xtream-recording` (main; `xtream-web` only with a web binary); `xtream-serve`, `xtream-cluster-agent` (node) |
| `/etc/nginx/sites-available/xtream.conf` | with `--with-nginx` |
| `/etc/sysctl.d/90-xtream.conf`, `/etc/security/limits.d/90-xtream.conf` | kernel and file-descriptor tuning |
| `/etc/postgresql/<ver>/main/conf.d/90-xtream.conf`, `/etc/redis/xtream.conf` | database and cache tuning sized from the machine's CPU and RAM |
| system user `xtream` | every service runs as this user, never as root |
| database | role `xtream`, database `xtreamprodb`, extensions `uuid-ossp` + `citext` |

The tuning files are rewritten on every run from `nproc` and `MemTotal` (re-run the installer after resizing the VM; `--no-tuning` skips it). PostgreSQL is restarted only when its tuning file changed. On a spinning disk, set `random_page_cost` back to `4` in the conf.d file if queries are slow.

---

## 6. Day-to-day operations

```sh
sudo systemctl status xtream-dashboard xtream-api xtream-worker xtream-serve
sudo journalctl -u xtream-api -f                       # one service
sudo journalctl -u 'xtream-*' --since -1h              # every service, last hour
sudo nano /etc/xtream/xtream.env                       # change the configuration …
sudo systemctl restart xtream-dashboard xtream-api xtream-worker xtream-serve   # … then restart
sudo -u xtream bash -c 'set -a; . /etc/xtream/xtream.env; /opt/xtream/bin/seed'  # migrate / seed again
xtream-serve sign "Film (2024)"                        # a signed /watch link for a test
```

Backups: the worker writes a verified `pg_dump` into `/var/lib/xtream/backups` every 24 hours (`DB_BACKUP_*` in the env file; `DB_BACKUP_ENCRYPTION_KEY` encrypts them, Settings → Backup copies them off-site to S3-compatible storage). By hand: `sudo -u postgres pg_dump -Fc xtreamprodb > xtream-$(date +%F).dump`, plus a tar of `/var/lib/xtream/media` and `/etc/xtream`.

---

## 7. Upgrading with `upgrade.sh`

`upgrade.sh` is in every release package. It downloads (or takes) the new package, verifies the checksum, **backs up** the binaries and `/etc/xtream` into `/opt/xtream/backup/<timestamp>/` (the last 3 are kept) and takes a pre-upgrade `pg_dump`, refuses to go on when the env file holds sample or too short secrets, calls `install.sh` with the role read from `/etc/xtream/serve.json` (env, secrets and configuration are kept), then runs a **health check** (dashboard `/healthz`, api `/readyz`, serve `/healthz`). A failure at any step **rolls back** to the previous binaries.

```sh
sudo ./upgrade.sh --tag v1.2.0          # download the package for this CPU from GitHub Releases + SHA256SUMS
sudo ./upgrade.sh --file xtreampro-v1.2.0-linux-amd64.tar.gz   # a downloaded package (verified when SHA256SUMS is next to it)
sudo ./upgrade.sh                       # from the extracted folder of the new package
sudo ./upgrade.sh --tag v1.2.0 --seed   # with the schema migration (when the release notes say so)
sudo ./upgrade.sh --rollback            # back to the newest backup
sudo ./upgrade.sh --tag v1.2.0 -- --with-recording   # options after "--" go to install.sh
```

An edge node upgrades the same way: `upgrade.sh` detects the `node` role, no URL or secret has to be given again. Nodes can also be upgraded from the dashboard (Servers → Install Load Balancer → **Upgrade**).

---

## 8. Removing with `uninstall.sh`

```sh
sudo ./uninstall.sh                 # stop + remove the units, binaries, tuning, nginx site; /etc/xtream archived to /root/xtream-config-<timestamp>.tgz
sudo ./uninstall.sh --purge-data    # also /var/lib/xtream, /var/cache/xtream and the xtream user
sudo ./uninstall.sh --drop-db       # also the database xtreamprodb and the role xtream
sudo ./uninstall.sh --all --yes     # everything, no questions
```

By default the media, recordings and the database are **kept**, so a re-install loses nothing. PostgreSQL, Redis, nginx, ufw and ffmpeg are never removed; certbot certificates stay in `/etc/letsencrypt`; the ufw rules for the 808x ports are removed, 22/80/443 stay.

---

## 9. Secrets and the database reset

### Sample secrets are refused

`dashboard`, `api` and `worker` exit at start when `JWT_SECRET` is unset, a sample value, or shorter than 32 bytes, or when `XTREAM_SECRET`, `CLUSTER_SYNC_TOKEN`, `SYSTEM_API_PASSWORD` or `ADMIN_API_KEY` is set to a sample value (`change-me…`, `mySecret`, `admin`, `password`, …). `upgrade.sh` checks `/etc/xtream/xtream.env` and `serve.json` before touching anything and stops naming the variable. Generate a value with `openssl rand -hex 32` (`CLUSTER_SYNC_TOKEN` must also match `sync_token` in `serve.json` and the setting `cluster.origin_sync_token`). `ALLOW_INSECURE_SECRETS=1` is for local development only.

### Reset the database to its freshly installed state: `reset.sh`

Deletes **all** panel data (users, lines, streams, films, settings, logs) and runs `seed` again (schema, default settings, the `admin` account). Media files on disk are not touched.

```sh
sudo ./reset.sh                  # systemd install: stop the services → pg_dump into /root → drop + recreate the database → seed → start the services
sudo ./reset.sh --yes --no-backup
```

After a reset **every dashboard password is gone**. The seed creates one `admin` account whose password is `ADMIN_PASSWORD` from `/etc/xtream/xtream.env`; when that is empty or a sample value, a random one is generated and printed **once** at the end (and must be changed at the first sign-in). The showcase accounts come back only with `--with-demo`. The script asks you to type the database name to confirm (`--yes` skips it) and flushes Redis database 0 (`--keep-redis` keeps it). To restore a dump: `pg_restore --clean --if-exists -d xtreamprodb <file>.dump`. An edge node has no database, so the script does not apply there.

---

## 10. Common errors

| Message / symptom | Cause | Fix |
| --- | --- | --- |
| `tar: Skipping to next header` / `Exiting with failure status` | corrupt `.tar.gz`: interrupted upload, or an HTML page downloaded instead of the file (link without `-L`, the release page instead of the asset) | `file xtreampro-*.tar.gz` must say `gzip compressed data` and `sha256sum -c SHA256SUMS --ignore-missing` must be `OK`; download again directly on the server (section 1) |
| `run as root` | `sudo` missing | `sudo ./install.sh …` |
| `binaries not found: missing …` | run from a folder without the binaries | `cd` into the extracted folder or pass `--bin-dir DIR` |
| `--web-domain: no public site binary (web) …` | a release package was installed with `--web-domain` | leave the option out, or build the site from source with `make build-web` and copy `build/web` next to the other binaries |
| `--main-url is required for --role node` | node options missing | see table B1 |
| `enroll: control plane answered 401 invalid_code` | wrong code, or older than 30 minutes | create a new code in the dashboard |
| `enroll: the control plane URL must be https://` | `--api-url` is `http://` | use `https://`, or `--allow-insecure` on a private network |
| Node answers `502 upstream unavailable` on VOD | the origin refuses the node (not in the enrolled list, or the origin has no `api_url`) | `sudo xtream-cluster doctor --origin https://media.example.com` on the node; check `api_url` in the origin's `serve.json` |
| Cluster Nodes shows **clock drift** / agent logs `clock is … off` | the node's clock is wrong | enable NTP (`timedatectl set-ntp true`); more than 60 s and requests are refused |
| Cluster Nodes shows **not reachable from outside** | the agent heartbeats, but `public_url` does not open from the main server | DNS, firewall, the node's certificate |
| The playback link from Lines → "Play link" returns 404 on the dashboard host | `get.php` is served by `api` (port 8082), not by the dashboard | set `API_PUBLIC_URL` to the API's public URL (the installer does this with `--api-domain`) and restart `xtream-dashboard` |
| `player_api.php` answers `Username or password is invalid` | the account is a panel **user**, not a **line** | create a line (Lines → Add Line) and take the play link from that row |
| `GeoIP … unavailable` in the API log | no `.mmdb` file yet | normal; the worker downloads one, or copy the files into `/var/lib/xtream/geoip` |
| The dashboard logs the proxy's address instead of the client's | nginx on another machine | put the proxy's address into `TRUSTED_PROXIES` in the env file |
| `upgrade.sh` stops with `… holds sample, empty or too short secrets` | the env file still carries sample values from an old install | generate new values (section 9) and re-run |
