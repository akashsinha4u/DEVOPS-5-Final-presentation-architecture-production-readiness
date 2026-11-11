# Single-Node Nomad + Consul Monitoring Platform

This repository contains the presentation and configuration examples for a single-node lab using Nomad, Consul, Docker, Prometheus, Grafana, Loki and Promtail.

## Repository structure

- `PRESENTATION.md` — full mentor presentation, architecture, design decisions, monitoring, troubleshooting, production-readiness notes and setup runbook.
- `config/` — Consul and Nomad agent configuration.
- `jobs/` — Nomad job specifications for Prometheus, Grafana, Loki and Promtail. Prometheus/Loki/Promtail application configuration is templated inside the relevant Nomad job files.
- `screenshots/` — illustrative UI mockups for layout/reference only.

## Important screenshot note

The two images in `screenshots/` are **illustrative mockups, not screenshots from a live deployment**. They are not proof that Nomad or Consul has been deployed. Replace them with genuine screenshots from your own running environment before presenting deployment evidence.

## Quick links

- [Full presentation](PRESENTATION.md)
- [Consul configuration](config/consul.hcl)
- [Nomad configuration](config/nomad.hcl)
- [Nomad jobs](jobs/)
- [UI reference images](screenshots/)

## Before running

Review `PRESENTATION.md`, especially the setup runbook and demo checklist. The lab settings are for a single-node learning environment; do not expose the UI/API ports publicly without appropriate network restrictions, ACLs and TLS.
