# Project Context — Open-Source Log Analysis & SIEM Lab

## Project Overview

This is a completed personal research and learning project. The goal was to evaluate and document a sustainable, open-source log analysis setup that is maintainable, reproducible, and deployable with Docker — built on existing platforms rather than from scratch.

Outcomes of this lab:

- Compared common open-source log analysis / SIEM-related tools and selected Wazuh as the primary platform
- Deployed Wazuh locally with Docker Compose (single-node)
- Ingested public sample datasets (SSH and Apache access logs)
- Wrote and validated custom detection rules for web scanning and login brute-force patterns

## Lab Setup

- Wazuh single-node stack via the official `wazuh-docker` repository (Docker Compose)
- Wazuh version used in this lab: 4.12.0
- Dashboard served locally over HTTPS (default Docker single-node ports)
- Credentials: use your own values; do not commit real passwords

For step-by-step setup, see [`Wazuh/WAZUH_BEGINNER_GUIDE.md`](../Wazuh/WAZUH_BEGINNER_GUIDE.md).

## Wazuh Docker Setup

Single-node deployment typically includes three containers:

- Wazuh manager — log analysis engine and threat detection
- Wazuh indexer — OpenSearch-based storage
- Wazuh dashboard — web UI

Container names may vary by Compose project name; confirm with `docker ps`.

### Useful Docker commands

```bash
# From the wazuh-docker single-node directory

# Start Wazuh
docker compose up -d

# Stop Wazuh
docker compose down

# Check container status
docker compose ps

# Copy a file into the manager container
docker cp <local_file> <manager_container_name>:<container_path>

# Run a command inside the manager container
docker exec -i <manager_container_name> <command>
```

## Tools Evaluated

| Tool | Category | Licence | Notes |
|---|---|---|---|
| Wazuh | SIEM / XDR | GPLv2 (free) | Primary platform for this project |
| ELK Stack | Log Management / SIEM | SSPL + Elastic License | Alternative to compare |
| Graylog | Log Management | SSPL | Complementary log management |
| Grafana + Loki | Observability | AGPLv3 | Performance / log observability |
| Security Onion | NSM / SIEM | GPLv2 | Possible future upgrade path |
| OpenObserve | Observability | AGPLv3 | Observability complement |
| Varonis | Data Security (commercial) | Proprietary | Compared for context only (not a SIEM substitute) |

**Key finding:** Wazuh was the best fit for this lab. Among the free options reviewed, it covers SIEM-style detection, FIM, vulnerability-related features, and compliance content in one stack. Commercial data-security platforms (e.g. Varonis) overlap more with UEBA / data access governance than with full SIEM replacement.

## Architecture

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

In this lab, no agents were deployed. Sample log files were ingested directly into the manager container for testing.

## Sample Datasets

### LogPAI Loghub — OpenSSH_2k.log

- About 2,000 lines of SSH logs
- Purpose: test detection around SSH login and brute-force style activity
- Typical path inside the manager: `/var/ossec/logs/OpenSSH_2k.log`

### AIT Log Data Set v1.1 — Apache access logs

- Public simulation dataset (Apache access log from a webmail-style service)
- Focus day used for testing: `2020-03-04`
- Ground-truth labels mark attack-related lines (benign vs attack types)
- Useful attack patterns in this lab:
  - **Nikto** — web vulnerability scanner; User-Agent contains `Nikto/`
  - **Hydra** — login brute-force / credential guessing; User-Agent contains `Mozilla/5.0 (Hydra)` and repeated hits on a login path

## Custom Detection Rules

Custom rules used in this lab live under [`Wazuh/`](../Wazuh/):

| Rule ID | Level | Match | Purpose |
|---|---|---|---|
| `100301` | 10 | `Nikto/` | Flag Nikto web scans in Apache access logs |
| `100302` | 10 | `Mozilla/5.0 (Hydra)` | Flag Hydra brute-force traffic in Apache access logs |

Implementation notes:

- Rules are children of parent rule `31108` (Apache/web related). Using the wrong parent (e.g. `31100`) prevents the custom rule from firing correctly in `wazuh-logtest`.
- Rules were validated with `wazuh-logtest`, then confirmed by replaying filtered log lines into a monitored Apache access-log path and checking Security events in the dashboard (`rule.id:100301` / `rule.id:100302`).
- Dashboard alert timestamps reflect ingest/replay time, not the original dates inside the historical logs.

See also [`Wazuh/local_rules.xml`](../Wazuh/local_rules.xml) and [`Wazuh/ait_lab_rules.xml`](../Wazuh/ait_lab_rules.xml).

## What Was Completed

1. Researched and compared several open-source log analysis / SIEM-related tools
2. Documented comparison findings and selected Wazuh as the primary platform
3. Deployed Wazuh single-node with Docker Compose
4. Ingested the LogPAI OpenSSH sample dataset into the Wazuh manager
5. Prepared an AIT Apache access-log slice for a single attack day and matched sample lines to ground-truth labels (Nikto, Hydra)
6. Wrote custom local rules for Nikto and Hydra User-Agent patterns
7. Validated detections with `wazuh-logtest` and dashboard replay

## Key Terminology

- **SIEM:** Security Information and Event Management — collect logs, detect threats, raise alerts
- **Agent:** Software on a monitored host that collects and forwards logs
- **Decoder:** Tells Wazuh how to parse a log format
- **Rule:** Defines suspicious behaviour and the alert to raise
- **FIM:** File Integrity Monitoring — detects file changes
- **UEBA:** User and Entity Behaviour Analytics — abnormal user/device behaviour
- **Telemetry pipeline:** Collects, filters, and routes logs before analysis (e.g. Fluent Bit, Logstash)

## Notes

- Wazuh Indexer = OpenSearch bundled with Wazuh
- Wazuh Dashboard = OpenSearch Dashboards with the Wazuh plugin
- OpenSearch is an Elasticsearch fork (Apache 2.0)
- SSPL (used by some ELK/Graylog offerings) is not OSI-approved open source
- This document describes a personal lab setup; it is not a production deployment guide
