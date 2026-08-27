# heimdall-k8s

[![Sponsor](https://img.shields.io/badge/Sponsor-%E2%9D%A4-ea4aaa?logo=githubsponsors&logoColor=white)](https://github.com/sponsors/johnycsf)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Issues](https://img.shields.io/badge/issues-welcome-lightgrey.svg)](../../issues/new/choose)

Heimdall on Kubernetes for homelab beginners — official PHP image, backup-before-update.

![`./manage.sh` control center](docs/manage-demo.gif)

## Install

```bash
git clone https://github.com/johnycsf/heimdall-k8s.git
cd heimdall-k8s
chmod +x manage.sh
./manage.sh
```

`./manage.sh` opens a **↑/↓ menu** with a `>` cursor (j/k and Enter also work). It asks for **StorageClass** and **replica count** (with a safe per-app suggestion). Re-run later to change those choices. Non-interactive: `STORAGE_CLASS=longhorn REPLICAS=1 ./manage.sh`.

Uses the **official** [`php:8.4-apache`](https://hub.docker.com/_/php) image and builds Heimdall from the [upstream release](https://github.com/linuxserver/Heimdall/releases) via the `Dockerfile` in this repo (no LinuxServer container runtime). `install.sh` builds `heimdall:local` and loads it into k3s/kind when those tools are present.

Docker Compose version (no Kubernetes needed): [heimdall-docker](https://github.com/johnycsf/heimdall-docker)

> **Updating an older clone?** Pulling git is safe. Re-running `./manage.sh` against a LinuxServer Deployment is not. Read [BREAKING-CHANGES.md](BREAKING-CHANGES.md).

## Why this repo (not just another manifest dump)

- **`./manage.sh`** control center — install, update, backup, status/doctor, uninstall
- Interactive colored install with step progress
- Auto-detects your OS and installs missing host tools (`kubectl`, `helm`, …)
- Choose **StorageClass** and **replica count** (re-run anytime to change)
- Safe **`./manage.sh update`** with automatic pre-update backup
- Incremental hardlink **`./manage.sh backup`** + restore
- **Official upstream images only**

## What you need

- A Kubernetes cluster (`kubectl` context already set)
- `sudo` on this machine so `./manage.sh` can install missing tools (kubectl, helm, curl, openssl, rsync, …)
- Disk for PersistentVolumes

## One-time: install Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace

kubectl -n longhorn-system get pod
```

Wait until the Longhorn pods are `Running` / `Ready`.
Longhorn will **automatically** create the Heimdall volume from the PVC — you do not need to create volumes by hand in the Longhorn UI.

## Open the dashboard

```bash
kubectl -n heimdall get svc heimdall
```

Use the `EXTERNAL-IP` (or your node IP with k3s ServiceLB / MetalLB):

- **HTTP:** `http://EXTERNAL-IP/`

Set `APP_URL` in `deploy.yaml` to that same URL after you know it, then re-apply.

## Customize

Edit `deploy.yaml` before installing (or re-apply after editing):

| Setting | Where | Notes |
|--------|--------|--------|
| Timezone | `TZ` | Default `America/New_York` |
| LAN app links | `ALLOW_INTERNAL_REQUESTS` | Set `true` so Heimdall can reach private IPs |
| Public URL | `APP_URL` | Match the URL you open in the browser |
| Disk size | PVC `storage` | Default `1Gi` |

## Update

Keep the stack current (safe while pods are running; brief rollout downtime):

```bash
./manage.sh update
```

Before changing anything, the script runs `./manage.sh backup` into `./backups` (incremental, database-safe). After a successful update it asks whether to **keep** or **delete** that snapshot, and how many local copies to retain (older ones are pruned). Copy important backups to an external drive, NAS, or cloud so they do not fill this disk.

This re-applies manifests, rolls Deployments so `:latest` images refresh, and prunes **unused** images on this machine when possible (k3s `crictl rmi --prune` or Docker dangling prune). PVCs and Secrets are left untouched.

Only for clusters already on `heimdall:local` — see [BREAKING-CHANGES.md](BREAKING-CHANGES.md).

## Backup and restore

Incremental snapshots via `rsync` hardlinks (unchanged files are not re-copied). `./manage.sh update` uses this same `backup.sh` before updating (into `./backups`).

```bash
# Backup to USB/NAS/external path (repeat anytime; later runs are incremental)
./manage.sh backup --dest /mnt/usb/heimdall-k8s-backups
./manage.sh backup --dest /mnt/usb/heimdall-k8s-backups --keep 5   # optional: retain only newest N

# On a brand-new machine/cluster after ./manage.sh:
./manage.sh backup --restore --from /mnt/usb/heimdall-k8s-backups
# or a local snapshot tree / specific snapshot:
./manage.sh backup --restore --from ./backups
./manage.sh backup --restore --from /mnt/usb/heimdall-k8s-backups/snapshots/YYYYMMDD-HHMMSS
```

Each snapshot includes `SHA256SUMS` plus a `snapshot_sha256` key in `META.txt`. Restore verifies these and **warns** (does not abort) if integrity is lost.

Keep the backup root on **one filesystem** so hardlinks work. Prefer an external drive, NAS, or cloud sync of that folder.

**Database safety:** Nextcloud uses a verified MariaDB *logical* dump (`mariadb-dump --single-transaction`) — the live `data/db` / DB PVC files are never rsync'd. SQLite apps (Heimdall, Vaultwarden) are stopped or scaled to 0, WAL-checkpointed when `sqlite3` is available, integrity-checked, then copied. Incremental hardlinks apply to file trees; each SQL dump is a full verified file with a SHA-256 in `META.txt`.

Older `backups/update-*` tarball folders (from previous script versions) are no longer used by `./manage.sh update`; use each folder's `RESTORE.txt` if you still need one, or delete them to free space.

## Uninstall

```bash
kubectl delete -f deploy.yaml
```

This also deletes the PVC and the Longhorn volume data. Or use **Uninstall** in `./manage.sh`.

## Notes for beginners

- One replica only — Heimdall config is not meant to be shared across many pods.
- Fresh install only — do not reuse a LinuxServer `/config` volume with this image.
- Multi-node clusters: push `heimdall:local` to a registry you control and update `image` / `imagePullPolicy` in `deploy.yaml`.
- Put Heimdall behind a reverse proxy (Traefik, nginx, Caddy) if you expose it outside your LAN.

## Backup exports

Local snapshots stay as incremental hardlink trees (fast rollback). Optionally create a compressed offsite copy with `./manage.sh backup --dest ./backups --archive tar.gz|tar.xz|zip` (add `--archive-password` for zip password or age-passphrase on tar). For stronger key-based encryption use `--encrypt` (age). See repo-framework `docs/BACKUP_ENCRYPTION.md`.

## Credits

This repo packages or configures upstream software. See [CREDITS.md](CREDITS.md) for the main developers and projects this work builds on.

## Disclaimer

This project is provided **as is**. The author is **not responsible** for any loss, damage, data corruption, downtime, security issues, or other consequences from using it. Full text: [DISCLAIMER.md](DISCLAIMER.md).

## Bug reports & contributions

If you hit an error, please [open a GitHub Issue](../../issues/new/choose) and follow [CONTRIBUTING.md](CONTRIBUTING.md). Fixes via Pull Request are welcome. GitHub Issues/PRs are the supported way to report problems—there is no private support channel.

## Security

See [SECURITY.md](SECURITY.md) for how to report vulnerabilities.

Sponsorship funds testing and maintenance: [github.com/sponsors/johnycsf](https://github.com/sponsors/johnycsf).
