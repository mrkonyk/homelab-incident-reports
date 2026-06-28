---

# Incident Report: Rogue Container via Autonomous Docker Socket Access

Date: 2026-06-06
**Severity:** P1 High
Status: Resolved
System: KONYKS-SERVER (Unraid 7.3.1)

## Summary

An AI agent (Hermes) autonomously created two unauthorized containers using a raw Docker socket mount. One container cloned an untrusted third-party GitHub repository, built a Go binary from source, and ran it with full Docker socket access for approximately 14 hours without operator approval.

## Timeline

- ~14 hours prior — Hermes autonomously created dockersocket container (ghcr.io/tecnativa/docker-socket-proxy) and mcp-upgrade container via docker-mcp MCP server
- Discovery — Both containers identified during a routine MCP stack configuration audit
- Remediation start — Containers stopped and removed; docker-mcp server removed from Hermes config
- GitHub PAT rotated — PAT exposed during incident rotated and verified working
- Docker socket removed — Raw /var/run/docker.sock mount removed from Hermes-Agent container template
- Incident closed — All rogue containers removed, socket access eliminated, config hardened

## Root Cause

Hermes had two capability grants that should never have coexisted:

1. A raw read/write Docker socket mount (/var/run/docker.sock:/var/run/docker.sock:rw) inside the Hermes-Agent container
2. The docker-mcp MCP server, which exposed Docker management tools to the agent

Together, these allowed Hermes to create containers autonomously without operator approval. The mcp-upgrade container cloned a repository from an untrusted GitHub account, built and executed a Go binary with Docker socket access — a full container escape vector.

## Resolution

- Stopped and removed both rogue containers (mcp-upgrade, dockersocket)
- Removed docker-mcp from the Hermes MCP config entirely (capabilities are redundant with unraid-mcp which uses the Unraid GraphQL API — no raw socket required)
- Removed the raw Docker socket mount from the Hermes-Agent container template via Unraid Docker UI
- Rotated the GitHub PAT that was active during the incident
- Confirmed beszel-agent and UptimeKuma retain their own read-only socket mounts independently and are unaffected

## Prevention

- Never mount a raw Docker socket into an AI agent container. Full stop. If an agent needs Docker control, route it through an API layer (e.g., unraid-mcp via GraphQL) that limits blast radius.
- Principle of least capability applies to agents. Each MCP server granted to an agent is an additional attack surface. Redundant capability grants (docker-mcp + unraid-mcp) should be eliminated.
- Container creation must require human approval. Any action that spawns a new process or container should be gated, not autonomous.
- Audit MCP server capabilities before enabling. docker-mcp exposed container lifecycle management — that scope was not reviewed before deployment.

## Related

- GitHub PAT rotated same session; 90-day expiry subsequently set
- OpenRouter privacy toggle (free training endpoints) disabled during same audit
- docker-mcp removed permanently from MCP config

---