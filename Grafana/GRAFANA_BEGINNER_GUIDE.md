# Grafana Observability Stack — Beginner Guide

This guide shows how to run a local Grafana + Loki + Prometheus lab with Docker, then explore sample logs in Explore.

## What each config file does

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Defines the containers, ports, volumes, and how services connect on `obs-net` |
| `loki-config.yml` | Loki settings: listen ports, filesystem storage, ingestion limits, schema |
| `prometheus.yml` | Prometheus scrape jobs (here: `node-exporter` every 15s) |
| `alloy/config.alloy` | Alloy pipelines: which log files to read, how to label/parse them, where to push (Loki) |
| `grafana/provisioning/datasources/loki.yml` | Auto-adds Loki as a Grafana datasource |
| `grafana/provisioning/datasources/prometheus.yml` | Auto-adds Prometheus as a Grafana datasource |

None of these files are “dashboards.” They wire the stack so Grafana can query logs (Loki) and metrics (Prometheus).

## Stack overview

```
Sample log files (./logs)
        ↓
Grafana Alloy (collector)
        ↓
Loki (log storage)          Prometheus ← node-exporter (host metrics)
        ↓                              ↓
              Grafana UI (:3000)
```

Services and ports:

- Grafana — `http://localhost:3000`
- Loki — `http://localhost:3100`
- Prometheus — `http://localhost:9090`
- Alloy UI — `http://localhost:12345`
- node-exporter — `http://localhost:9100`

Default Grafana login for this lab: `admin` / `admin` (change for anything beyond local practice).

## 1. Prerequisites

- Docker Desktop installed and running
- Docker Compose available
- A few GB of free disk/RAM for the containers

## 2. Prepare sample logs

Do **not** commit large datasets into git. Place files locally under `Grafana/logs/`:

```text
Grafana/logs/
  OpenSSH_2k.log                 # optional: LogPAI OpenSSH sample
  samples/OpenSSH_2k.log         # optional alternate path
  mail.insect.com/               # optional: AIT Log Data Set v1 host folder
    apache2/
    auth.log
    ...
```

Public datasets used in this lab:

- LogPAI Loghub — OpenSSH sample
- AIT Log Data Set v1.1 — `mail.insect.com` logs

Alloy is already configured to read:

- `/var/log/samples/*.log` (maps to `./logs/*.log`)
- `/var/log/samples/mail.insect.com/**/*` (maps to `./logs/mail.insect.com/...`)

It skips heavy/binary artifacts such as `eve.json` and `audit.zip`.

## 3. Start the stack

From this `Grafana/` directory:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

## 4. Open Grafana and query logs

1. Open `http://localhost:3000`
2. Log in with `admin` / `admin`
3. Go to **Explore**
4. Choose datasource **Loki**
5. Try a LogQL query, for example:
   - `{job="logs"}` — root sample logs (e.g. OpenSSH)
   - `{job="ait-mail-insect"}` — AIT mail.insect.com stream
   - `{job="ait-mail-insect", log_type="apache2"}` — Apache access/error only

For metrics, switch datasource to **Prometheus** and browse targets or use a simple query such as `up`.

## 5. Stop and clean up

```bash
docker compose down
```

To remove containers and anonymous volumes as well:

```bash
docker compose down -v
```

## 6. Troubleshooting

- **No log lines in Explore:** confirm files exist under `./logs`, wait ~10–30s after Alloy starts, and widen the time range (AIT logs are dated ~2020).
- **Datasource errors:** ensure Loki/Prometheus containers are healthy (`docker compose ps` / `docker compose logs`).
- **Old timestamps:** Alloy parses timestamps from many AIT log formats so Explore can jump to Feb/Mar 2020; if a format is unmatched, Loki may use ingest time instead.

## Summary

1. Put sample logs in `./logs` (not in git).
2. `docker compose up -d` from `Grafana/`.
3. Open Grafana → Explore → Loki (and optionally Prometheus).
4. Query by `job` / `log_type` labels.

This is a local observability lab. Wazuh (under `../Wazuh/`) is the SIEM/detection side of the same overall project.
