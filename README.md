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
- Severity (P0 Critical / P1 High / P2 Medium / P3 Low)
- Summary (one-line description)
- Timeline (what happened and when)
- Root Cause
- Resolution
- Prevention / Follow-up

## Reports Index
| Date | Severity | Title | Status |
|------|----------|-------|--------|
| 2026-06-06 | P1 High | Rogue Container via Autonomous Docker Socket Access | Resolved |
| 2026-06-10 | P2 Medium | XPS 13 Boot Failure — Broken initramfs After Kernel Update | Resolved |
| 2026-06-11 | P1 High | CrowdSec Silent No-Op + Unauthenticated LAN Service Exposure | Resolved |
| 2026-06-11 | P0 Critical | Nextcloud 500 — Hardcoded DB Host After Container Recreation | Resolved |
| 2026-06-11 | P1 High | Full-Stack Credential Rotation Triggered by a Hardcoded-IP Outage | Resolved |
| 2026-06-11 | P2 Medium | Docker Network Segmentation — Three-Phase Zero-Downtime Migration | Resolved |
| 2026-06-11 | P3 Low | Containers Showing "3rd Party" After Network Segmentation Migration | Resolved |
| 2026-06-16 | P3 Low | Proactive Docker Audit — Stale Network References and Orphaned Templates | Resolved |
| 2026-06-17 | P2 Medium | Backup Strategy Hardening: Skip List Audit, Beszel WAL Fix, and First Restore Test | Resolved |
| 2026-06-19 | P2 Medium | GitOps Foundation: Webhook Limitation, Silent Polling Failure, and Observability Stack Hardening | Resolved |
| 2026-06-20 | P1 High | Icon Label Change Surfaces Three Latent Infrastructure Gaps | Resolved |
| 2026-06-20 | P2 Medium | Appdata Backup Plugin Cannot Resolve Bind-Backed Named Docker Volumes | Resolved |
| 2026-06-20 | P2 Medium | Komodo Silently Dropped a Queued Deploy After Periphery Restart | Resolved |
| 2026-06-20 | P2 Medium | Nextcloud Database Connectivity Loss — Dropped Secondary Network + Stale PHP-FPM DNS Cache | Resolved |
| 2026-06-21 | P1 High | Multi-System Failures During Routine Maintenance | Resolved |
| 2026-06-23 | P2 Medium | GitHub PAT Rotation Script: Believed-Staged Rollout Was Actually a Single Unscoped Write Across 24 Stacks | Resolved |
| 2026-06-26 | P0 Critical | CrowdSec LAPI Key Exposure — Detection, Rotation, and Incomplete-Purge Discovery | Resolved |
| 2026-06-26 | P1 High | Edge Auth Stack Remediation — Closing 17 Audit Findings and Three Independent Silent-Failure Discoveries | Resolved |
| 2026-06-26 | P2 Medium | External Heartbeat Monitoring: Dead-Man's-Switch Coverage for Three Single Points of Failure | Complete |
| 2026-06-27 | P2 Medium | Centralized Secrets Migration — Five Independent Verification Failures Caught Before Any Reached Production | Resolved |
| 2026-06-29 | P1 High | Credential Rotation Script Bugs — Three Compounding Failures Render do_push_rotation.sh Inoperable | Resolved |
| 2026-06-29 | P1 High | Ghost Cron Trigger Causes Silent Multi-Database Backup Outage | Resolved |
| 2026-06-29 | P1 High | Grafana Admin Password Misconfiguration — Secret Mount Was a Directory, Password Reset on Every Restart | Resolved |
| 2026-06-29 | P2 Medium | Hermes-Agent Cron Fleet Audit — Weekly IP Scan Running Clean While Primary Scan Mechanism Had Never Executed | Resolved |
| 2026-06-29 | P2 Medium | Weekly IP Scan — Two Independent MCP Failure Classes Found in Same Script | Resolved |
| 2026-06-29 | P1 High / P1 High / P2 Medium | Authelia Secrets Rotation — Three Incidents: Redis Credential Exposure, CACHE_URL Colon-Stripping, SHA256 Newline Artifact | Resolved |
| 2026-07-02 | P0 Critical | Credential Exposure During Cron Audit and Remediation | Structurally Resolved |
| 2026-07-02 | P1 High | Silent Monitoring Failure: Backup Verification Never Actually Checked Anything | Structurally Resolved |
| 2026-07-03 | P2 Medium | KomodoDeployLag on observability/seerr — A PAT-Rotation Race, Not a Repeat of Prior Deploy-Drop Bugs | Resolved |
| 2026-07-03 | P1 High | Silent MCP Toolset Misconfiguration Caused Weeks of Fabricated Monitoring Reports | Resolved |
| 2026-07-04 | P1 High | Stale CLI Version, GitOps Near-Miss, and Cascading Credential Exposure | Resolved |
| 2026-07-04 | P1 High | Webhook Chain Re-Verification, an Accidental Credential Exposure, and a Self-Caused Credential Path Divergence | Resolved |

## Methodology
Infrastructure is treated as code. Changes are documented, tested, and reviewed before application. AI-assisted tooling (Hermes + Claude Code) is used for complex multi-system operations with human approval at each step.

---
*Maintained by a Network Analyst in training. Thunder Bay, ON.*
