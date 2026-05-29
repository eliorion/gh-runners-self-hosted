# gh-runners-self-hosted

Ansible project to provision a Debian VM as self-hosted GitHub Actions runners, with a Docker registry pull-through cache and an S3-compatible build cache.

## What gets provisioned

| Service | Port | Description |
|---------|------|-------------|
| `apt-cacher-ng` | 3142 | APT package cache |
| `registry-mirror` | 5000 | Docker pull-through cache (registry:2) |
| `minio` | 9000 / 9001 | S3-compatible GHA tool cache (console on 9001) |
| `github-runner-runner-N` | — | N systemd-managed GitHub Actions runner processes |

All services run as systemd units and restart automatically on failure.

## Prerequisites

- Debian VM reachable over the network (LAN or Tailscale)
- SSH access as a user with `sudo` or `su` to root
- Python 3 on the VM (`./bootstrap` installs it if missing)
- SSH agent running locally with your key loaded (`ssh-add`)
- GitHub classic PAT with `repo` scope (or fine-grained with `Administration: write`)

## Setup

**1. Copy and fill in `.env`:**

```bash
cp .env.example .env
$EDITOR .env
```

| Variable | Required | Description |
|----------|----------|-------------|
| `VM_HOST` | yes | IP or hostname of the Debian VM |
| `VM_USER` | yes | SSH user (must have sudo after first run) |
| `SSH_PORT` | no | SSH port, default `22` |
| `RUNNER_SCOPE` | yes | `repo` for a single repo, `org` for an org |
| `GITHUB_REPO` | if scope=repo | `owner/repo` |
| `GITHUB_ORG` | if scope=org | org name |
| `RUNNER_COUNT` | no | Number of parallel runners, default `2` |
| `GITHUB_TOKEN` | yes | PAT (can also be set in shell env) |
| `MINIO_ACCESS_KEY` | no | S3 cache access key, default `minioadmin` |
| `MINIO_SECRET_KEY` | no | S3 cache secret key, default `minioadmin` |

**2. First run** (sets up sudo, creates user):

```bash
./bootstrap --first-run   # prompts for root password via su
```

**3. Subsequent runs:**

```bash
./bootstrap
```

Re-running is idempotent. Changing `GITHUB_REPO` and re-running re-registers the runners automatically.

## Local caches

### Docker registry mirror

Docker is configured to pull images through a local registry:2 cache at `127.0.0.1:5000`. First pull hits Docker Hub; subsequent pulls are served from disk at `/var/lib/registry`.

No workflow changes needed, no credentials, no endpoint to configure — the mirror is fully transparent. The Docker daemon routes all pulls through it automatically via `/etc/docker/daemon.json`.

### GitHub Actions tool cache (MinIO)

MinIO runs at `127.0.0.1:9000` and provides an S3-compatible cache endpoint. Tool installs (Node, Python, Go, etc.) via `actions/setup-*` are cached in `/opt/hostedtoolcache` and shared across all runs on the VM.

Credentials are set in `.env` (`MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`) and injected into every runner job automatically via the systemd unit — no per-workflow config needed. The runner environment exposes:

```
ACTIONS_CACHE_URL=http://127.0.0.1:9000/
AWS_ACCESS_KEY_ID=<MINIO_ACCESS_KEY>
AWS_SECRET_ACCESS_KEY=<MINIO_SECRET_KEY>
```

Change `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY` in `.env` before first run, then re-run `./bootstrap` to apply.

## Customisation

Key variables are in `ansible/group_vars/all.yaml`. Override any at runtime:

```bash
./bootstrap -e runner_count=4
```

Runner labels default to `self-hosted,linux,x64,debian`.

## Troubleshooting

Check service status on the VM:

```bash
ssh sysadmin@<VM_HOST> "systemctl status github-runner-runner-1 registry-mirror minio"
```

View runner logs:

```bash
ssh sysadmin@<VM_HOST> "journalctl -u github-runner-runner-1 -f"
```
