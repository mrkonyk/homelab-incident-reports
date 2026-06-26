# Homelab Incident Reports

A living record of infrastructure incidents, audits, and resolutions from KONYKS-SERVER — a self-hosted homelab operated with SRE principles.

## About This Repository
This repo documents real incidents encountered while running a production-grade self-hosted stack. Each report follows a structured format: problem identification, root cause analysis, resolution, and prevention measures.

## Stack Overview
- Unraid 7.3.1 on Lenovo M900 Tiny (i7-6700T, 32GB RAM)
- 29 Docker containers across tiered network segments
- Services: Nextcloud, Immich, Authelia, SWAG, CrowdSec, Home Assistant
- Monitoring: Beszel, Uptime Kuma, Stack Health Dashboard (T0-T6)
- AI layer: Hermes agent with MCP integrations

## Incident Report Format
Each report includes:
- Date & Duration
- Severity (Critical / Warning / Info)
- Summary (one-line description)
- Timeline (what happened and when)
- Root Cause
- Resolution
- Prevention / Follow-up

## Reports Index
| Date | Severity | Title | Status |
|------|----------|-------|--------|
| 2026-06-11 | Critical | Nextcloud 500 — Hardcoded DB Host After Container Recreation | Resolved |
| 2026-06-11 | Warning | Containers Showing 3rd Party After Network Migration | Resolved |
| 2026-06-11 | Critical | Full Stack Credential Rotation After Audit | Resolved |
| 2026-06-11 | Critical | Docker Network Segmentation — 3-Phase Migration | Resolved |
| 2026-06-26 | Critical | CrowdSec LAPI Key Exposure — Detection, Rotation, and Incomplete-Purge Discovery | Resolved |

## Methodology
Infrastructure is treated as code. Changes are documented, tested, and reviewed before application. AI-assisted tooling (Hermes + Claude Code) is used for complex multi-system operations with human approval at each step.

---
*Maintained by a Network Analyst in training. Thunder Bay, ON.*
