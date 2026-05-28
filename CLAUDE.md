# gh-runners-self-hosted

Ansible project to provision a Debian VM for self-hosted GitHub Actions runners.

## Stack

- Control node: devcontainer (Ubuntu 24.04), Ansible installed via mise
- Target: Debian VM on Proxmox, reachable via Tailscale subnet route (Tailscale on Proxmox host only)
- SSH: agent forwarding via `SSH_AUTH_SOCK` — no key path needed

## Entry point

```bash
./bootstrap --first-run   # first time: su escalation (root password)
./bootstrap               # subsequent runs: sudo NOPASSWD
```

Config in `.env` (copy from `.env.example`).

## Ansible

All files use `.yaml` extension. Inventory generated at runtime into `ansible/inventory/hosts.yaml` (gitignored).

### Role order (`ansible/site.yaml`)

1. `apt_cache` — must run first; removes dead proxy config if apt-cacher-ng not running
2. `common` — installs sudo, creates sysadmin, passwordless sudo
3. `docker` — Docker CE from official repo
4. `docker_registry_mirror` — registry:2 pull-through cache on 127.0.0.1:5000
5. `gha_cache` — MinIO on 127.0.0.1:9000, `/opt/hostedtoolcache`
6. `github_runner` — downloads runner binary, registers N runners, systemd services

### Key variables (`ansible/group_vars/runners.yaml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `runner_user` | `sysadmin` | OS user for runner processes |
| `runner_count` | `2` | Override via `.env` `RUNNER_COUNT` |
| `registry_mirror_port` | `5000` | Docker pull-through cache port |
| `apt_cache_port` | `3142` | apt-cacher-ng port |
| `minio_port` | `9000` | MinIO S3 API port |

### Runner scopes

- `RUNNER_SCOPE=repo` + `GITHUB_REPO=owner/repo` — single repo runner
- `RUNNER_SCOPE=org` + `GITHUB_ORG=myorg` — org-wide runner

GitHub token: classic PAT with `repo` scope required.

Changing `GITHUB_REPO` and rerunning `./bootstrap` auto-reconfigures runners (detects URL mismatch in `.runner` file).

## Containers (systemd-managed)

All containers run via systemd units (no Docker Compose), managed as `Type=simple` with `Restart=on-failure`.
Units: `registry-mirror.service`, `minio.service`, `github-runner-runner-{N}.service`
