# Open-Source Log Analysis & SIEM Lab

## Project Overview

This is a completed personal research and learning project. The goal was to evaluate and document a sustainable, open-source log analysis setup that is maintainable, reproducible, and deployable with Docker — built on existing platforms rather than from scratch.

The lab has two complementary tracks:

- **Wazuh** — SIEM-style detection, custom rules, alert validation
- **Grafana stack** — observability (Loki logs + Prometheus metrics) for exploring the same kinds of sample datasets

Outcomes of this lab:

- Compared common open-source log analysis / SIEM-related tools
- Deployed Wazuh locally with Docker Compose (single-node) and wrote custom detection rules
- Deployed a Grafana + Loki + Alloy + Prometheus stack and explored sample logs in Grafana Explore
- Practiced with public sample datasets (SSH and Apache / host logs)

## Repo layout

| Path | Contents |
|------|----------|
| [`Wazuh/`](Wazuh/) | Beginner guide + custom local rules (Nikto / Hydra) |
| [`Grafana/`](Grafana/) | Docker Compose observability stack + beginner guide |

## Lab guides

- Wazuh setup: [`Wazuh/WAZUH_BEGINNER_GUIDE.md`](Wazuh/WAZUH_BEGINNER_GUIDE.md)
- Grafana stack setup: [`Grafana/GRAFANA_BEGINNER_GUIDE.md`](Grafana/GRAFANA_BEGINNER_GUIDE.md)

## Tools Evaluated

| Tool | Category | Licence | Notes |
|---|---|---|---|
| Wazuh | SIEM / XDR | GPLv2 (free) | Primary SIEM / detection platform for this project |
| Grafana + Loki (+ Alloy / Prometheus) | Observability | AGPLv3 (Grafana/Loki) | Hands-on log + metrics lab (see `Grafana/`) |
| ELK Stack | Log Management / SIEM | SSPL + Elastic License | Alternative to compare |
| Graylog | Log Management | SSPL | Complementary log management |
| Security Onion | NSM / SIEM | GPLv2 | Possible future upgrade path |
| OpenObserve | Observability | AGPLv3 | Observability complement |
| Varonis | Data Security (commercial) | Proprietary | Compared for context only (not a SIEM substitute) |

**Key finding:** Wazuh was the best fit for detection / SIEM-style work. Grafana + Loki is a strong companion for log search and metrics dashboards, but it is not a drop-in replacement for Wazuh’s rule engine and security alerting. Commercial data-security platforms (e.g. Varonis) overlap more with UEBA / data access governance than with full SIEM replacement.

## Architecture

### Wazuh (detection)

```
Log sources (hosts, apps, network devices)
        ↓
Wazuh Agent (optional; installed on monitored machines)
        ↓
Wazuh Manager (threat detection, rule matching)
        ↓
Wazuh Indexer (OpenSearch — storage & search)
        ↓
Wazuh Dashboard (web UI — alerts, dashboards)
```

In the Wazuh lab phase, no agents were deployed. Sample log files were ingested directly into the manager container for testing.

### Grafana stack (observability)

```
Sample log files (./Grafana/logs)
        ↓
Grafana Alloy (collector / parser)
        ↓
Loki (log storage)          Prometheus ← node-exporter
        ↓                              ↓
              Grafana UI (Explore / dashboards)
```

## Wazuh Docker Setup

Single-node deployment typically includes three containers:

- Wazuh manager — log analysis engine and threat detection
- Wazuh indexer — OpenSearch-based storage
- Wazuh dashboard — web UI

Container names may vary by Compose project name; confirm with `docker ps`.

Version used in this lab: **Wazuh 4.12.0**. Dashboard is served locally over HTTPS (default Docker single-node ports). Use your own credentials; do not commit real passwords.

### Useful Docker commands

```bash
# From the wazuh-docker single-node directory

docker compose up -d
docker compose down
docker compose ps
docker cp <local_file> <manager_container_name>:<container_path>
docker exec -i <manager_container_name> <command>
```

## Grafana stack setup

The observability lab lives under [`Grafana/`](Grafana/) and includes:

- **Loki** — log storage
- **Alloy** — reads local sample logs and pushes them to Loki
- **Prometheus** + **node-exporter** — host metrics
- **Grafana** — UI on `http://localhost:3000`

Quick start (details in the beginner guide):

```bash
cd Grafana
# Place sample logs under ./logs (not committed to git)
docker compose up -d
# Open http://localhost:3000  (lab default: admin / admin)
```

Config files in that folder (`docker-compose.yml`, `loki-config.yml`, `prometheus.yml`, `alloy/config.alloy`, Grafana datasource provisioning) define how the containers run and connect — not the dashboards themselves.

## Sample Datasets

### LogPAI Loghub — OpenSSH_2k.log

- About 2,000 lines of SSH logs
- Used with Wazuh (manager ingest) and optionally with Grafana/Loki via Alloy
- Purpose: explore SSH login / brute-force style activity

### AIT Log Data Set v1.1 — Apache / host logs

- Public simulation dataset (`mail.insect.com` and related logs)
- Focus day used for Wazuh rule testing: `2020-03-04`
- Useful attack patterns:
  - **Nikto** — web vulnerability scanner; User-Agent contains `Nikto/`
  - **Hydra** — login brute-force; User-Agent contains `Mozilla/5.0 (Hydra)`
- Also mounted into the Grafana stack under `Grafana/logs/mail.insect.com/` for Explore queries

Large raw log folders are **not** stored in this repository. Download datasets locally and place them where each guide describes.

## Custom Detection Rules (Wazuh)

Custom rules live under [`Wazuh/`](Wazuh/):

| Rule ID | Level | Match | Purpose |
|---|---|---|---|
| `100301` | 10 | `Nikto/` | Flag Nikto web scans in Apache access logs |
| `100302` | 10 | `Mozilla/5.0 (Hydra)` | Flag Hydra brute-force traffic in Apache access logs |

Implementation notes:

- Rules are children of parent rule `31108` (Apache/web related). Using the wrong parent (e.g. `31100`) prevents the custom rule from firing correctly in `wazuh-logtest`.
- Rules were validated with `wazuh-logtest`, then confirmed by replaying filtered log lines into a monitored Apache access-log path and checking Security events in the dashboard (`rule.id:100301` / `rule.id:100302`).
- Dashboard alert timestamps reflect ingest/replay time, not the original dates inside the historical logs.

See also [`Wazuh/local_rules.xml`](Wazuh/local_rules.xml) and [`Wazuh/ait_lab_rules.xml`](Wazuh/ait_lab_rules.xml).

## What Was Completed

1. Researched and compared several open-source log analysis / SIEM-related tools
2. Documented comparison findings; selected Wazuh for detection and Grafana/Loki for observability practice
3. Deployed Wazuh single-node with Docker Compose
4. Ingested the LogPAI OpenSSH sample dataset into the Wazuh manager
5. Prepared an AIT Apache access-log slice and matched sample lines to ground-truth labels (Nikto, Hydra)
6. Wrote and validated custom local rules with `wazuh-logtest` and dashboard replay
7. Deployed Grafana + Loki + Alloy + Prometheus and explored sample logs in Grafana Explore

## Key Terminology

- **SIEM:** Security Information and Event Management — collect logs, detect threats, raise alerts
- **Observability stack:** Tools for storing and exploring logs/metrics (here: Loki, Prometheus, Grafana)
- **Agent:** Software on a monitored host that collects and forwards logs
- **Decoder:** Tells Wazuh how to parse a log format
- **Rule:** Defines suspicious behaviour and the alert to raise
- **FIM:** File Integrity Monitoring — detects file changes
- **UEBA:** User and Entity Behaviour Analytics — abnormal user/device behaviour
- **Telemetry pipeline:** Collects, filters, and routes logs before analysis (e.g. Alloy, Fluent Bit, Logstash)
- **LogQL:** Loki query language used in Grafana Explore for log search

## Notes

- Wazuh Indexer = OpenSearch bundled with Wazuh; Wazuh Dashboard = OpenSearch Dashboards with the Wazuh plugin
- OpenSearch is an Elasticsearch fork (Apache 2.0)
- SSPL (used by some ELK/Graylog offerings) is not OSI-approved open source
- Grafana lab default password in Compose is for local learning only
- This repository documents a personal lab setup; it is not a production deployment guide
