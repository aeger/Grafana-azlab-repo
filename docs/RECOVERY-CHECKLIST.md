# Recovery Checklist — AZ-Lab Monitoring

Use this when rebuilding `svc-podman-01` or restoring monitoring after a failure.

## 0. Sanity
- OS boots
- Network up
- DNS resolving internal + public names
- Time synced (chrony/systemd-timesyncd)

## 1. Base packages
```bash
sudo apt update
sudo apt install -y podman podman-compose curl jq ca-certificates dos2unix
sudo loginctl enable-linger "$USER"
```

## 2. Networking
```bash
podman network exists proxy || podman network create proxy
```

## 3. Restore repo
```bash
cd ~
git clone <REPO_URL> monitoring
cd monitoring
```
or restore from backup.

## 4. Secrets
- Restore `grafana.env`
- Verify permissions:
```bash
chmod 600 grafana.env
```

## 5. Restore data (optional)
```bash
rsync -a /backup/prometheus-data/ ./prometheus/data/
rsync -a /backup/grafana-data/ ./grafana/data/
```

## 6. Validate configs
```bash
dos2unix compose.yml prometheus/config/*.yml

podman run --rm -v "$PWD/prometheus/config:/etc/prometheus:Z"   docker.io/prom/prometheus:latest   promtool check config /etc/prometheus/prometheus.yml
```

## 7. Start stack
```bash
podman compose up -d
podman ps
```

## 8. Verify
- Prometheus targets UP
- Blackbox probes returning expected results
- SNMP metrics visible
- Grafana dashboards load

## 9. Common failures
- **401 on Traefik** → expected
- **TLS errors on .home.arpa** → use insecure module or mount CA
- **No SNMP data** → check firewall + community + module name

## 10. Done
Monitoring restored.
