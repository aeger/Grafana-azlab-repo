# AZ-Lab Monitoring (Prometheus + Grafana + Exporters) — Rootless Podman

This repo contains the **monitoring stack** used in my AZ‑Lab homelab on `svc-podman-01`:

- Prometheus
- Grafana
- Blackbox Exporter (HTTP/S probes)
- SNMP Exporter (MikroTik modules)
- Node Exporter
- cAdvisor

It’s designed for **rootless Podman** + **podman-compose**, with an emphasis on “recoverable, repeatable, no-clickops”.  
If you’re reading this during a recovery: sorry. You’ll be back online soon.

## Repo layout

```
monitoring/
  compose.yml
  grafana.env.example
  grafana/                 # (Grafana provisioning / optional)
  prometheus/
    config/
      prometheus.yml
      blackbox.yml
      snmp.yml             # generated (snmp_exporter generator output)
      generator.yml        # input for generator (do not edit snmp.yml manually)
    data/                  # Prometheus TSDB (ignored)
```

> **Line endings policy:** configs must be **UTF‑8, LF**, no BOM.  
> Before deploying, run `dos2unix` and validate configs against the running images.

## Prereqs (Ubuntu on svc-podman-01)

Install Podman + podman-compose and basic tools:

```bash
sudo apt update
sudo apt install -y podman podman-compose curl ca-certificates jq dos2unix
```

Enable lingering so user services can run after logout (recommended):

```bash
sudo loginctl enable-linger "$USER"
```

Create the network used by the stack (adjust if you already have one):

```bash
podman network exists proxy || podman network create proxy
```

## First-time setup

### 1) Clone the repo

```bash
mkdir -p ~/monitoring
cd ~
git clone <YOUR_GITHUB_REPO_URL> monitoring
cd ~/monitoring
```

### 2) Configure Grafana env

Copy the example and fill in values:

```bash
cp -a grafana.env.example grafana.env
chmod 600 grafana.env
dos2unix grafana.env
```

**Do not commit** `grafana.env`.

### 3) Validate config files (recommended)

```bash
dos2unix compose.yml prometheus/config/*.yml

# Prometheus config lint (inside the image)
podman run --rm -v "$PWD/prometheus/config:/etc/prometheus:Z"   docker.io/prom/prometheus:latest   promtool check config /etc/prometheus/prometheus.yml
```

### 4) Start the stack

```bash
podman compose up -d
podman ps
```

Open Grafana (via your reverse proxy / ingress) and confirm:
- Prometheus target is up
- Blackbox probes show expected results (401 for protected endpoints is fine if you configured it that way)
- SNMP exporter is returning interface metrics

## Recovery procedure (fresh server / rebuild)

This is the “my server died and I want my dashboards back” section.

1. Install prereqs (Podman, podman-compose, dos2unix)
2. Restore DNS/ingress so `grafana.<domain>` is reachable (Traefik stack lives elsewhere)
3. Clone this repo to `~/monitoring`
4. Restore `grafana.env` (from your password manager / backup)
5. Restore persistent data (optional but nice)
   - Prometheus: `prometheus/data/`
   - Grafana: whatever volume/path you mount in `compose.yml`
6. Validate configs with `promtool`
7. `podman compose up -d`

### Restoring data (optional)

If you backed up these folders, copy them back before starting:

```bash
# Prometheus TSDB
rsync -a --delete /backup/prometheus-data/ ~/monitoring/prometheus/data/

# Grafana data (depends on your compose mounts)
rsync -a --delete /backup/grafana-data/ ~/monitoring/grafana/data/
```

Then bring everything up:

```bash
cd ~/monitoring
podman compose up -d
```

## SNMP: how `snmp.yml` is managed

`prometheus/config/snmp.yml` is **generated**. Manual edits are overwritten.

- `generator.yml` is the source
- `snmp.yml` is the output

To regenerate (only when changing modules/auth/OIDs):

```bash
cd ~/monitoring/prometheus/config

# Example: run the generator image and write snmp.yml
podman run --rm -v "$PWD:/opt:Z"   docker.io/prom/snmp-generator:latest   generate   -o /opt/snmp.yml   -c /opt/generator.yml

dos2unix snmp.yml
```

## Blackbox: internal TLS and protected endpoints

Internal `.home.arpa` endpoints commonly use an internal CA or self-signed certs.  
If you see `x509: certificate signed by unknown authority`, you have two options:

- **Best:** mount your internal CA into blackbox and use `tls_config.ca_file`
- **Fast:** use a module with `tls_config.insecure_skip_verify: true` for internal targets

Protected dashboards (like Traefik) typically return **401** without auth.  
That can still be treated as “up” using `valid_status_codes: [200,401,403]` for those targets.

## Common commands

```bash
# Logs
podman logs -f prometheus
podman logs -f blackbox
podman logs -f snmp_exporter

# Restart stack
podman compose down
podman compose up -d

# Validate Prometheus config
podman run --rm -v "$PWD/prometheus/config:/etc/prometheus:Z"   docker.io/prom/prometheus:latest   promtool check config /etc/prometheus/prometheus.yml
```

## Git hygiene

- Keep secrets out of Git.
- Keep configs LF/UTF‑8 (use `dos2unix`).
- Prefer edits + validation **before** restart/redeploy.

See also: my ingress stack patterns in `aeger/traefik-rootless`.
