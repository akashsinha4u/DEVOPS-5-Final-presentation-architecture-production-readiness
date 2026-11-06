# Single-Node Nomad + Consul Platform with Prometheus, Grafana and Loki

30-minute mentor presentation: architecture, decisions, monitoring, troubleshooting, production readiness.

> **Scope:** one VM running Consul, Nomad (server + client in one agent), Docker, and four Nomad jobs: Prometheus, Grafana, Loki, Promtail.
> **Principle:** a simple working setup beats a complex broken one.

---

## Table of Contents

1. [Repo layout](#1-repo-layout)
2. [Architecture overview (5 min)](#2-architecture-overview-5-min)
3. [Deployment decisions (5 min)](#3-deployment-decisions-5-min)
4. [Monitoring strategy (5 min)](#4-monitoring-strategy-5-min)
5. [Troubleshooting process (5 min)](#5-troubleshooting-process-5-min)
6. [Production readiness (5 min)](#6-production-readiness-5-min)
7. [Q&A preparation (5 min)](#7-qa-preparation-5-min)
8. [Setup runbook (reproduce from scratch)](#8-setup-runbook)
9. [Demo checklist](#9-demo-checklist)

---

## 1. Repo layout

```
.
├── PRESENTATION.md          <- this file
├── config/
│   ├── consul.hcl
│   ├── nomad.hcl
│   ├── prometheus.yml.tpl   (embedded in job, shown below)
│   ├── loki.yml
│   └── promtail.yml
└── jobs/
    ├── prometheus.nomad.hcl
    ├── grafana.nomad.hcl
    ├── loki.nomad.hcl
    └── promtail.nomad.hcl
```

The configs are written out in full in sections 3 and 8. Copy each block into the file named in its heading.

---

## 2. Architecture overview (5 min)

### Diagram

```
                         ┌──────────────────────── Single VM ─────────────────────────┐
                         │                                                            │
   Browser ──────────────┼─► Grafana :3000 ──► Prometheus :9090 ──► (scrapes)         │
   (you / mentor)        │        │                  ▲  ▲  ▲                          │
                         │        │                  │  │  └── Consul :8500 metrics   │
                         │        ▼                  │  └───── Nomad  :4646 metrics   │
                         │     Loki :3100 ◄── Promtail (reads alloc log files)        │
                         │                           │                                │
                         │     ┌─────────────────────┴──────────────────┐             │
                         │     │ Consul :8500  (service catalog + DNS)  │             │
                         │     └───────────▲────────────────────────────┘             │
                         │                 │ registers services + health checks       │
                         │     ┌───────────┴────────────────────────────┐             │
                         │     │ Nomad agent  (server + client) :4646   │             │
                         │     │   scheduler ─► Docker driver ─► tasks  │             │
                         │     └────────────────────────────────────────┘             │
                         └────────────────────────────────────────────────────────────┘
```

### Roles

| Component | Role |
|---|---|
| **Nomad** | Scheduler and orchestrator. Takes a job spec, places allocations, runs them via the Docker driver, restarts and reschedules on failure. |
| **Consul** | Service catalog, health checking, and DNS (`<name>.service.consul`). Also a KV store. |
| **Docker** | Container runtime used by Nomad's `docker` task driver. |
| **Prometheus** | Pulls metrics. Targets are found dynamically through Consul, not static IPs. |
| **Grafana** | Dashboards. Reads from Prometheus (metrics) and Loki (logs). |
| **Loki** | Log storage and query (LogQL). |
| **Promtail** | Log shipper. Runs as a `system` job, reads Nomad allocation log files, pushes to Loki. |

### Service discovery flow (say this out loud)

1. I run `nomad job run prometheus.nomad.hcl`.
2. The Nomad **server** creates an evaluation, and the scheduler picks a client with enough free CPU and memory. That produces an **allocation**.
3. The Nomad **client** starts the Docker container.
4. The job has a `service {}` block with a `check {}`. Nomad registers that service with **Consul**, and Consul runs the health check.
5. Prometheus has `consul_sd_configs`. It asks Consul's catalog "which services exist?" and scrapes each healthy target. **No hardcoded IPs.** If an allocation moves and gets a new port, Prometheus follows automatically.
6. Grafana reads Prometheus and Loki through the datasource config. Other apps can reach services by DNS, e.g. `loki.service.consul`.

### Log flow

Task stdout/stderr is written by Nomad to `/opt/nomad/data/alloc/<alloc_id>/alloc/logs/<task>.stdout.0`. Promtail tails those files, extracts `alloc_id`, `task` and `stream` labels from the filename, and pushes to Loki. Grafana queries Loki.

---

## 3. Deployment decisions (5 min)

### 3.1 Resource allocation

Lab VM: 2 vCPU, 4 GB RAM (about 4000 MHz CPU shares, about 3.5 GB usable by Nomad).

| Job | CPU (MHz) | Memory (MB) | Why |
|---|---|---|---|
| Prometheus | 500 | 512 | TSDB and query memory grows with series count. Few targets in a lab, so 512 MB is enough. |
| Grafana | 200 | 256 | Mostly idle, spikes on dashboard load. |
| Loki | 300 | 512 | Ingest buffers and index cache. |
| Promtail | 100 | 128 | Lightweight tailer. |
| **Total** | **1100** | **1408** | About 35% of the node, leaving headroom for Nomad, Consul, Docker and the OS. |

Decisions:
- **Always set `cpu` and `memory` explicitly.** Nomad schedules on these numbers. Without them, it uses defaults and the bin-packing is meaningless.
- **Leave headroom (about 30 to 40%)** for the OS, the Nomad and Consul agents, and Docker.
- **`memory_max`** (oversubscription) on Prometheus and Loki lets them burst without a hard OOM kill at the reserved value. This needs memory oversubscription enabled in the scheduler config.
- **Never schedule 100% of the node.** A noisy neighbour would starve the agents and the node would look "down".

### 3.2 Networking: host mode

I use `network { mode = "host" }` with static ports.

- Prometheus needs to reach Consul at `127.0.0.1:8500` and Nomad at `127.0.0.1:4646`. In bridge mode, `127.0.0.1` is the container itself, and that is the most common bug here (see troubleshooting).
- Promtail to Loki and Grafana to Prometheus/Loki are simple on localhost.
- **Trade-off:** host mode has no port isolation and needs static ports (conflict risk). In production I would use bridge mode plus Consul Connect (sidecar proxies), or dynamic ports with Consul DNS.

### 3.3 Persistence: host volumes

Prometheus, Loki and Grafana data live in Nomad **host volumes**, so a restart or reschedule on this node does not lose data.

- Declared in the Nomad client config, claimed in the job with `volume` and `volume_mount`.
- **Permissions matter.** Containers run as different UIDs: Prometheus `65534` (nobody), Grafana `472`, Loki `10001`. The host directories must be `chown`ed accordingly.
- Limit: host volumes tie data to one node. Production uses CSI volumes or object storage (S3) instead.

### 3.4 Job stanzas and why

| Stanza | Setting | Reason |
|---|---|---|
| `restart` | `attempts=3, interval=5m, delay=15s, mode=delay` | Handle transient failures locally without flapping forever. |
| `reschedule` | `delay=30s, unlimited=true` | If the node/alloc is lost, try elsewhere. Single node here, but correct for scale-out. |
| `update` | `max_parallel=1, health_check="checks", min_healthy_time=10s, auto_revert=true` | Rolling deploys that **roll back automatically** if the new version is unhealthy. |
| `service` + `check` | HTTP check on `/-/healthy`, `/api/health`, `/ready` | Consul only advertises healthy instances, and Nomad uses check results for deployments. |
| `template` | Renders configs from inside the job | Config is versioned with the job. `change_mode = "signal"` reloads without restarting. |
| `type` | `service` for the stack, `system` for Promtail | `system` runs one copy on every client node, which is what a log agent needs. |

### 3.5 Configuration files

#### `config/consul.hcl`

```hcl
datacenter = "dc1"
data_dir   = "/opt/consul/data"
node_name  = "lab-node-1"

server           = true
bootstrap_expect = 1            # single node. Production: 3 or 5

bind_addr   = "127.0.0.1"
client_addr = "0.0.0.0"         # lab only; restrict in production

ui_config {
  enabled = true
}

telemetry {
  prometheus_retention_time = "60s"
  disable_hostname          = true
}
```

Run: `consul agent -config-file=config/consul.hcl`

#### `config/nomad.hcl`

```hcl
datacenter = "dc1"
data_dir   = "/opt/nomad/data"
bind_addr  = "0.0.0.0"

server {
  enabled          = true
  bootstrap_expect = 1          # single node. Production: 3 or 5
}

client {
  enabled = true

  host_volume "prometheus" {
    path      = "/opt/volumes/prometheus"
    read_only = false
  }
  host_volume "grafana" {
    path      = "/opt/volumes/grafana"
    read_only = false
  }
  host_volume "loki" {
    path      = "/opt/volumes/loki"
    read_only = false
  }
}

consul {
  address = "127.0.0.1:8500"    # Nomad registers services here
}

plugin "docker" {
  config {
    volumes {
      enabled = true            # allow bind mounts (Promtail needs this)
    }
    allow_privileged = false
  }
}

telemetry {
  collection_interval        = "5s"
  prometheus_metrics         = true
  publish_allocation_metrics = true
  publish_node_metrics       = true
}
```

Run: `sudo nomad agent -config=config/nomad.hcl`

#### `jobs/prometheus.nomad.hcl`

```hcl
job "prometheus" {
  datacenters = ["dc1"]
  type        = "service"

  update {
    max_parallel     = 1
    health_check     = "checks"
    min_healthy_time = "10s"
    healthy_deadline = "3m"
    auto_revert      = true
  }

  group "prometheus" {
    count = 1

    network {
      mode = "host"
      port "http" { static = 9090 }
    }

    restart {
      attempts = 3
      interval = "5m"
      delay    = "15s"
      mode     = "delay"
    }

    volume "prometheus" {
      type      = "host"
      source    = "prometheus"
      read_only = false
    }

    service {
      name = "prometheus"
      port = "http"
      tags = ["metrics"]

      check {
        type     = "http"
        path     = "/-/healthy"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "prometheus" {
      driver = "docker"

      config {
        image        = "prom/prometheus:v2.53.0"
        network_mode = "host"
        args = [
          "--config.file=/etc/prometheus/prometheus.yml",
          "--storage.tsdb.path=/prometheus",
          "--storage.tsdb.retention.time=15d",
          "--web.enable-lifecycle",
          "--web.listen-address=0.0.0.0:9090",
        ]
        volumes = [
          "local/prometheus.yml:/etc/prometheus/prometheus.yml",
          "local/alerts.yml:/etc/prometheus/alerts.yml",
        ]
      }

      volume_mount {
        volume      = "prometheus"
        destination = "/prometheus"
      }

      template {
        destination   = "local/prometheus.yml"
        change_mode   = "signal"
        change_signal = "SIGHUP"
        data          = <<-EOT
        global:
          scrape_interval: 15s
          evaluation_interval: 15s

        rule_files:
          - /etc/prometheus/alerts.yml

        scrape_configs:
          # Prometheus scrapes itself
          - job_name: prometheus
            static_configs:
              - targets: ["127.0.0.1:9090"]

          # Nomad metrics, discovered through Consul (Nomad registers itself as "nomad")
          - job_name: nomad
            metrics_path: /v1/metrics
            params:
              format: [prometheus]
            consul_sd_configs:
              - server: "127.0.0.1:8500"
                services: ["nomad"]
            relabel_configs:
              - source_labels: [__meta_consul_service_metadata_external_source]
                regex: ""
                action: drop
              - source_labels: [__meta_consul_tags]
                regex: .*,http,.*
                action: keep
              - source_labels: [__meta_consul_node]
                target_label: instance

          # Consul's own metrics
          - job_name: consul
            metrics_path: /v1/agent/metrics
            params:
              format: [prometheus]
            static_configs:
              - targets: ["127.0.0.1:8500"]

          # Any app that registers with the "metrics" tag is picked up automatically
          - job_name: consul-services
            consul_sd_configs:
              - server: "127.0.0.1:8500"
                tags: ["metrics"]
            relabel_configs:
              - source_labels: [__meta_consul_service]
                target_label: service
        EOT
      }

      template {
        destination = "local/alerts.yml"
        change_mode = "signal"
        change_signal = "SIGHUP"
        data          = <<-EOT
        groups:
          - name: platform
            rules:
              - alert: TargetDown
                expr: up == 0
                for: 2m
                labels: { severity: critical }
                annotations:
                  summary: "{{ "{{ $labels.job }}" }} on {{ "{{ $labels.instance }}" }} is down"

              - alert: DiskAlmostFull
                expr: nomad_client_host_disk_used_percent > 85
                for: 5m
                labels: { severity: warning }
                annotations:
                  summary: "Disk above 85% on {{ "{{ $labels.instance }}" }}"

              - alert: ConsulNoLeader
                expr: consul_raft_leader == 0
                for: 1m
                labels: { severity: critical }
                annotations:
                  summary: "Consul has no Raft leader"

              - alert: AllocationOOM
                expr: increase(nomad_client_allocs_oom_killed[10m]) > 0
                labels: { severity: warning }
                annotations:
                  summary: "A Nomad allocation was OOM-killed"
        EOT
      }

      resources {
        cpu        = 500
        memory     = 512
        memory_max = 1024
      }
    }
  }
}
```

> Note: the `{{ "{{ ... }}" }}` escaping is needed because Nomad templates also use `{{ }}` syntax. If a metric name differs on your Nomad version, check `curl -s localhost:4646/v1/metrics?format=prometheus | grep <keyword>` and adjust the expression.

#### `jobs/loki.nomad.hcl`

```hcl
job "loki" {
  datacenters = ["dc1"]
  type        = "service"

  group "loki" {
    network {
      mode = "host"
      port "http" { static = 3100 }
    }

    volume "loki" {
      type   = "host"
      source = "loki"
    }

    service {
      name = "loki"
      port = "http"
      tags = ["metrics"]
      check {
        type     = "http"
        path     = "/ready"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "loki" {
      driver = "docker"

      config {
        image        = "grafana/loki:3.1.0"
        network_mode = "host"
        args         = ["-config.file=/etc/loki/loki.yml"]
        volumes      = ["local/loki.yml:/etc/loki/loki.yml"]
      }

      volume_mount {
        volume      = "loki"
        destination = "/loki"
      }

      template {
        destination = "local/loki.yml"
        data        = <<-EOT
        auth_enabled: false

        server:
          http_listen_port: 3100
          grpc_listen_port: 9096

        common:
          instance_addr: 127.0.0.1
          path_prefix: /loki
          storage:
            filesystem:
              chunks_directory: /loki/chunks
              rules_directory: /loki/rules
          replication_factor: 1
          ring:
            kvstore:
              store: inmemory

        schema_config:
          configs:
            - from: 2024-01-01
              store: tsdb
              object_store: filesystem
              schema: v13
              index:
                prefix: index_
                period: 24h

        limits_config:
          retention_period: 168h        # 7 days
          reject_old_samples: true
          reject_old_samples_max_age: 168h

        compactor:
          working_directory: /loki/compactor
          retention_enabled: true
          delete_request_store: filesystem
        EOT
      }

      resources {
        cpu        = 300
        memory     = 512
        memory_max = 768
      }
    }
  }
}
```

#### `jobs/promtail.nomad.hcl`

```hcl
job "promtail" {
  datacenters = ["dc1"]
  type        = "system"          # one per client node

  group "promtail" {
    network {
      mode = "host"
      port "http" { static = 9080 }
    }

    service {
      name = "promtail"
      port = "http"
      tags = ["metrics"]
      check {
        type     = "http"
        path     = "/ready"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "promtail" {
      driver = "docker"

      config {
        image        = "grafana/promtail:3.1.0"
        network_mode = "host"
        args         = ["-config.file=/etc/promtail/promtail.yml"]
        volumes = [
          "local/promtail.yml:/etc/promtail/promtail.yml",
          "/opt/nomad/data/alloc:/alloc-data:ro",
        ]
      }

      template {
        destination = "local/promtail.yml"
        data        = <<-EOT
        server:
          http_listen_port: 9080
          grpc_listen_port: 0

        positions:
          filename: /tmp/positions.yaml

        clients:
          - url: http://127.0.0.1:3100/loki/api/v1/push

        scrape_configs:
          - job_name: nomad-alloc-logs
            static_configs:
              - targets: [localhost]
                labels:
                  job: nomad-logs
                  __path__: /alloc-data/*/alloc/logs/*.std*.*
            pipeline_stages:
              - regex:
                  source: filename
                  expression: '/alloc-data/(?P<alloc_id>[^/]+)/alloc/logs/(?P<task>[^.]+)\.(?P<stream>stdout|stderr)\.'
              - labels:
                  alloc_id:
                  task:
                  stream:
        EOT
      }

      resources {
        cpu    = 100
        memory = 128
      }
    }
  }
}
```

> Promtail is in maintenance mode upstream, and Grafana Alloy is its successor. I used Promtail because it is simple and well documented. In production I would migrate to Alloy.

#### `jobs/grafana.nomad.hcl`

```hcl
job "grafana" {
  datacenters = ["dc1"]
  type        = "service"

  group "grafana" {
    network {
      mode = "host"
      port "http" { static = 3000 }
    }

    volume "grafana" {
      type   = "host"
      source = "grafana"
    }

    service {
      name = "grafana"
      port = "http"
      check {
        type     = "http"
        path     = "/api/health"
        interval = "10s"
        timeout  = "2s"
      }
    }

    task "grafana" {
      driver = "docker"

      config {
        image        = "grafana/grafana:11.1.0"
        network_mode = "host"
        volumes = [
          "local/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml",
        ]
      }

      volume_mount {
        volume      = "grafana"
        destination = "/var/lib/grafana"
      }

      env {
        GF_SERVER_HTTP_PORT        = "3000"
        GF_AUTH_ANONYMOUS_ENABLED  = "false"
        # Lab only. In production inject via Vault, never hardcode:
        GF_SECURITY_ADMIN_PASSWORD = "change-me"
      }

      template {
        destination = "local/datasources.yml"
        data        = <<-EOT
        apiVersion: 1
        datasources:
          - name: Prometheus
            type: prometheus
            access: proxy
            url: http://127.0.0.1:9090
            isDefault: true
          - name: Loki
            type: loki
            access: proxy
            url: http://127.0.0.1:3100
        EOT
      }

      resources {
        cpu    = 200
        memory = 256
      }
    }
  }
}
```

Datasources are **provisioned as code**, so there is no clicking and the setup is reproducible.

---

## 4. Monitoring strategy (5 min)

### What I monitor and why

I follow the **USE method** (Utilization, Saturation, Errors) for infrastructure and **RED** (Rate, Errors, Duration) for services.

| Dashboard row | Metrics | Why it matters |
|---|---|---|
| **Node health** | CPU %, memory used/available, disk used %, network I/O | If the node is saturated, everything on it suffers. Disk full is the classic silent killer for Prometheus and Loki. |
| **Nomad scheduling** | Allocations running/failed/pending, restarts, allocated vs available CPU/memory | Shows capacity pressure *before* jobs get stuck as "pending: resources exhausted". |
| **Allocation usage** | Per-task CPU and memory vs the limit I set | Validates the resource decisions in section 3. If usage is at 95% of the limit, I am under-provisioned. If at 10%, I am wasting capacity. |
| **Consul** | `consul_raft_leader`, registered service count, health checks passing/critical | A missing leader means the control plane is broken. Critical checks mean a service is unhealthy. |
| **Monitoring stack self-health** | `up` per target, Prometheus TSDB size and series count, scrape duration, Loki ingestion rate | The monitoring system must itself be monitored. |
| **Logs (Loki)** | Log volume per task, error-rate panel, live log panel | Metrics say *that* something is wrong, logs say *why*. |

### Example queries

```promql
# Targets that are down
up == 0

# Memory used vs allocated per task (Nomad)
nomad_client_allocs_memory_usage / nomad_client_allocs_memory_allocated

# Consul has a leader (1 = yes)
consul_raft_leader

# Prometheus TSDB head series (cardinality watch)
prometheus_tsdb_head_series
```

```logql
# Error rate by task over 5 minutes
sum by (task) (rate({job="nomad-logs"} |= "error" [5m]))

# Live logs for one task
{job="nomad-logs", task="prometheus"}
```

> Metric names vary slightly between Nomad versions. Before the demo, open `http://localhost:4646/v1/metrics?format=prometheus` and confirm the names you use on the dashboard exist.

### Alerting (3 rules, and why these)

1. **TargetDown** (`up == 0` for 2m): the most basic signal that something stopped.
2. **DiskAlmostFull** (>85%): the disk fills slowly, so the alert gives time to act.
3. **ConsulNoLeader**: the control plane is down and no new service registration can happen.

Plus **AllocationOOM**, which catches mis-sized memory limits. Production routes these through **Alertmanager** to Slack or PagerDuty. I defined the rules but did not build the routing in the lab.

---

## 5. Troubleshooting process (5 min)

### Methodology (the part to emphasise)

1. **Define the symptom precisely.** What is failing, since when, what changed?
2. **Work top-down.** Job, then evaluation, then allocation, then task, then logs.
3. **Form one hypothesis, test it, change one thing at a time.**
4. **Find the root cause, not just the workaround.**
5. **Verify the fix and record it** so it is not rediscovered later.

### Command ladder

```bash
nomad job status <job>                      # is it running / pending / dead?
nomad eval status <eval-id>                 # why couldn't it be placed?
nomad alloc status <alloc-id>               # task events: Received, Failed, Restarting
nomad alloc logs -stderr <alloc-id> <task>  # what the app said
nomad alloc fs <alloc-id>                   # inspect files in the allocation
consul members ; consul catalog services    # is the platform itself healthy?
curl -s localhost:8500/v1/health/state/critical   # failing checks
journalctl -u nomad -u consul -e            # agent-level problems
```

### Issue write-ups

> **Before presenting:** reproduce at least one of these yourself and replace the "Evidence" lines with your real terminal output/screenshots. The mentor will ask follow-ups, so present the one you personally hit and understood. These are the failures this exact setup commonly produces.

#### Issue A: Loki crash-loops with `permission denied`

- **Symptom:** the `loki` allocation keeps restarting, and the job shows `Restarting` then `Failed`.
- **Investigation:**
  1. `nomad job status loki` shows repeated restarts.
  2. `nomad alloc status <id>` shows the task exits with code 1 right after start.
  3. `nomad alloc logs -stderr <id> loki` shows `mkdir /loki/chunks: permission denied`.
- **Root cause:** the host volume `/opt/volumes/loki` was created by root. The Loki container runs as **UID 10001**, so it cannot write.
- **Fix:** `sudo chown -R 10001:10001 /opt/volumes/loki`, then `nomad job run jobs/loki.nomad.hcl`.
- **Verification:** `curl localhost:3100/ready` returns `ready`, and the Consul check turns passing.
- **Prevention:** document the UID of each image next to the volume setup (Prometheus 65534, Grafana 472, Loki 10001), and create the directories with the right owner in the provisioning script.

#### Issue B: job stuck in `pending`

- **Symptom:** `nomad job status` shows the allocation `pending` and a "Placement Failure".
- **Investigation:** `nomad eval status <id>` shows `Resources exhausted on 1 nodes: Dimension "memory" exhausted`.
- **Root cause:** the sum of memory requested by all tasks exceeded what the node had available (I had set a task to 2048 MB on a 4 GB VM).
- **Fix:** right-size the request based on observed usage in Grafana, rather than guessing, and use `memory_max` for headroom.
- **Lesson:** the scheduler is working correctly. The error was in my capacity plan.

#### Issue C: Grafana shows "connection refused" to Prometheus

- **Symptom:** the datasource test fails.
- **Root cause:** an earlier version used bridge networking and `http://localhost:9090`. Inside the container, `localhost` is the container itself.
- **Fix:** use host networking (as now) or Consul DNS / the host IP.
- **Lesson:** the networking mode determines what `localhost` means.

#### Issue D: Consul health check fails but the app is "running"

- **Symptom:** the task is up, but Consul shows the check as critical, so the service is not discoverable and Prometheus drops the target.
- **Root cause:** the app was bound to `127.0.0.1` while the check hit the advertised address (or the wrong path or port).
- **Fix:** bind to `0.0.0.0` (or the correct interface) and verify the path with `curl`.
- **Lesson:** "running" and "healthy" are different, and discovery depends on health.

---

## 6. Production readiness (5 min)

### 6.1 What must change

| Area | Lab (now) | Production |
|---|---|---|
| **Availability** | 1 node, `bootstrap_expect=1` | **3 or 5 Consul and Nomad servers** (Raft quorum, tolerates 1 or 2 failures) across availability zones. **Separate server nodes from client nodes.** Multiple clients so workloads can reschedule. |
| **Data placement** | Host volumes | Object storage (S3) for Loki chunks. Prometheus with `remote_write` to long-term storage (Thanos, Mimir, or Cortex). CSI volumes for stateful jobs. |
| **Networking** | Host mode, static ports | Bridge mode plus **Consul Connect** (service mesh with mTLS), private subnets, load balancer in front of Grafana. |
| **Alerting** | Rules only | Alertmanager with routing to Slack or PagerDuty, silences, and runbooks linked in alerts. |
| **Config management** | Manual files | **Terraform** (infra) + **Packer** (images) + CI pipeline that runs `nomad job validate` and `nomad job plan` before deploy. Everything in Git. |
| **Capacity** | Guessed | Based on measured usage, with autoscaling (Nomad Autoscaler). |
| **Upgrades** | None | Rolling upgrades (servers first, one at a time), with `update` stanza and `auto_revert`. |
| **Retention / limits** | 15d metrics, 7d logs | Defined per compliance and cost needs. Log rotation configured on clients. |

### 6.2 Backup and disaster recovery

| Asset | Backup method | Frequency |
|---|---|---|
| Nomad state (jobs, ACLs, evals) | `nomad operator snapshot save backup.snap` | Every 1h by cron/systemd timer |
| Consul state (catalog, KV, ACLs) | `consul snapshot save backup.snap` | Every 1h |
| Job specs and configs | Git (source of truth) | On every change |
| Metrics (Prometheus) | `remote_write` to durable storage, or TSDB snapshot API | Continuous |
| Logs (Loki) | S3 object store with versioning and lifecycle policy | Continuous |
| Grafana dashboards | Provisioned from Git (dashboards as code), DB backup for the rest | On every change |

Rules:
- Snapshots are **encrypted** and copied **off the node** (S3 in a different region or account).
- **A backup that has never been restored is not a backup.** I would schedule a restore drill (quarterly) into a fresh environment and measure how long it takes.
- Define targets: for example **RPO 1 hour** (matches snapshot interval) and **RTO 30 minutes** (rebuild from Terraform/Packer, restore snapshots, redeploy jobs from Git).
- Because the jobs are declarative and in Git, the real recovery of *workloads* is "re-run the jobs". Snapshots mainly protect the cluster state and ACLs.

**DR scenarios:**
1. *One server dies:* the cluster keeps quorum, so replace the node and join it.
2. *Quorum lost:* use the documented Raft recovery (`peers.json` for Consul, `nomad operator raft` tooling) or restore the latest snapshot.
3. *Region/AZ lost:* rebuild with Terraform in another region, restore snapshots, redeploy from Git, and point DNS to the new load balancer.

### 6.3 Security improvements

Current lab state is **intentionally open and not secure**. For production:

1. **ACLs** enabled on both Consul and Nomad (default-deny), with least-privilege tokens per job and per human.
2. **TLS everywhere:** mTLS for Nomad RPC/HTTP and Consul, plus gossip encryption keys.
3. **Secrets in Vault**, injected into jobs through the `template` or `vault` stanza. No passwords in job files (the Grafana password in my lab file is a placeholder I would remove).
4. **Service mesh (Consul Connect)** with intentions, so only allowed services can talk to each other.
5. **Network:** private subnets, security groups or firewall allowing only required ports, no public Consul/Nomad API (bind to private interfaces, `client_addr` restricted).
6. **Grafana:** TLS, SSO (OIDC), disable anonymous access, and role-based access.
7. **Containers:** run as non-root, pinned image versions (done) or digests, image scanning, `allow_privileged = false` (done), read-only mounts where possible (Promtail mount is read-only).
8. **Audit logging** and OS hardening, patching, and SSH key or SSM-only access.

---

## 7. Q&A preparation (5 min)

**Why Nomad instead of Kubernetes?**
Nomad is a single binary with a small operational footprint, and it schedules containers, VMs and raw executables. It integrates natively with Consul and Vault. Kubernetes has a bigger ecosystem and is the better choice for large teams that need that tooling. For a small platform, Nomad's simplicity wins.

**What is the biggest risk of this setup?**
Single node means it is a single point of failure. Node loss means total outage and the data on host volumes is at risk. The fix is covered in 6.1 and 6.2.

**What happens if Consul goes down?**
Running allocations keep running, because they do not depend on Consul at runtime. But service registration, health checking, DNS discovery and Prometheus's `consul_sd` refresh stop working (Prometheus keeps scraping its last known targets for a while). With 3+ servers, one failure does not cause this.

**What happens if Nomad goes down?**
Already running containers continue, but there is no scheduling, rescheduling or restart-on-failure management until it is back.

**Why Consul service discovery for Prometheus instead of static targets?**
Targets change as allocations are rescheduled. Consul gives the current healthy set automatically, with no manual config edits.

**How do you handle secrets?**
Not in job files. Vault with Nomad's integration, or at minimum Nomad Variables (built-in, ACL-protected). Not built in the lab.

**How would you scale this?**
Add more client nodes (Nomad bin-packs across them), 3 server nodes, and move state to shared or object storage. Add the Nomad Autoscaler for demand-based scaling.

**How do you know your resource limits are right?**
By measurement. The allocation usage panel compares real usage with the limit, so I adjust from data.

**Why Promtail when it is deprecated?**
Simple, well documented, and fine for the lab. Alloy is the migration path.

**What would you do first with one more week?**
Add the 3-node server cluster with ACLs and TLS, then Alertmanager routing and a tested restore.

---

## 8. Setup runbook

Assumes Ubuntu 22.04/24.04 on a VM with 2 vCPU / 4 GB RAM. Commands run as a user with sudo.

```bash
# 1. Docker
sudo apt-get update && sudo apt-get install -y docker.io
sudo systemctl enable --now docker

# 2. HashiCorp repo + Consul + Nomad
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y consul nomad

# 3. Directories + correct ownership for container UIDs
sudo mkdir -p /opt/consul/data /opt/nomad/data /opt/volumes/{prometheus,grafana,loki}
sudo chown -R 65534:65534 /opt/volumes/prometheus
sudo chown -R 472:472     /opt/volumes/grafana
sudo chown -R 10001:10001 /opt/volumes/loki

# 4. Start Consul, then Nomad (separate terminals, or turn into systemd units)
sudo consul agent -config-file=config/consul.hcl
sudo nomad agent  -config=config/nomad.hcl

# 5. Verify the platform
consul members
nomad server members
nomad node status

# 6. Deploy the stack (order: storage first, then consumers)
export NOMAD_ADDR=http://127.0.0.1:4646
nomad job validate jobs/loki.nomad.hcl       # validate every file first
nomad job run jobs/loki.nomad.hcl
nomad job run jobs/prometheus.nomad.hcl
nomad job run jobs/promtail.nomad.hcl
nomad job run jobs/grafana.nomad.hcl

# 7. Verify
nomad job status
consul catalog services
curl -s localhost:9090/api/v1/targets | head
curl -s localhost:3100/ready
```

Access points:

| Service | URL |
|---|---|
| Nomad UI | http://localhost:4646 |
| Consul UI | http://localhost:8500 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 (admin / the password set in the job) |
| Loki | http://localhost:3100/ready |

---

## 9. Demo checklist

Before the call:
- [ ] All 4 jobs show `running` in `nomad job status`
- [ ] Prometheus **Status > Targets** is all green
- [ ] Grafana dashboard shows live data, and the Loki logs panel is populated
- [ ] Browser tabs open: Nomad UI, Consul UI, Prometheus targets, Grafana dashboard
- [ ] Terminal with large font, `NOMAD_ADDR` exported
- [ ] One troubleshooting issue **reproduced and its real output saved**
- [ ] Backup commands tested at least once: `nomad operator snapshot save` and `consul snapshot save`
- [ ] Grafana dashboard JSON exported and committed to the repo

Timing:

| Min | Section |
|---|---|
| 0:00 to 5:00 | Architecture (diagram, then show Consul UI with registered services) |
| 5:00 to 10:00 | Deployment decisions (walk one job file, show resource numbers) |
| 10:00 to 15:00 | Monitoring (live Grafana dashboard, then a Loki query) |
| 15:00 to 20:00 | Troubleshooting (walk the command ladder on your real issue) |
| 20:00 to 25:00 | Production readiness (HA, backup/DR, security) |
| 25:00 to 30:00 | Q&A |

**Remember:** talk through *why*, not just *what*. Where something is not built, say "I haven't built this, and here is how I would approach it". The brief explicitly allows that.
