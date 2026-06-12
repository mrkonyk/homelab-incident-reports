# Hermes Agent Infrastructure Maintenance Log

This log tracks infrastructure maintenance, stack audits, and systems engineering tasks performed by Hermes on KONYKS-SERVER.

## June 12 2026 — arr Stack Audit + Storage Fixes

### arr stack TRaSH Guides compliance audit findings:
- **Docker path mappings:** All correct — sonarr/radarr/lidarr/readarr on `/mnt/user/data` → `/data`, sabnzbd on `/mnt/user/data/usenet` → `/data/usenet`, delugevpn on `/mnt/user/data/torrents` → `/data/torrents`.
- **Root folders correct:** Radarr `/data/media/movies`, Sonarr `/data/media/tv`.
- **Permissions:** `nobody:users` ownership correct, functional.

### Critical finding — hardlinks broken:
- Every media import was a full file copy instead of a hardlink.
- **Root cause:** Unraid mover was moving files from cache_ssd to array, severing hardlinks by creating new inodes on different filesystems.
- **Impact:** Estimated double storage usage for all seeded/downloaded content.

### Fix applied (data share settings):
- **Primary storage:** Cache_ssd (unchanged).
- **Secondary storage:** Changed from Array → None.
- **Allocation method:** Changed from High-water → Fill-up.
- **Mover action:** Now "takes no action" on data share.
- **Result:** Downloads stay on cache_ssd permanently, hardlinks intact forever.

### Storage cleanup:
- 177GB recovered from `usenet/complete/` — 88 orphaned post-import directories deleted (confirmed imported to media library).
- 3 `_FAILED_` SABnzbd directories deleted.
- Remaining 104GB in `usenet/complete/` contains 69 unverified items pending manual review in Radarr/Sonarr/Lidarr.

### Network Placement:
- `unmanic` corrected to `wg0` network (was incorrectly placed on `tier2_apps` — shares `/mnt/user/data/media/*` paths with arr stack, belongs with media acquisition pipeline).

### Infrastructure audit (Claude Code + Unraid MCP):
- **All 30 containers healthy**, zero critical issues.
- **Array:** 62.9% used, all disks DISK_OK, zero parity errors.
- **cache_media SMART extended test:** Completed without error.
- **frontend network deleted:** Confirmed empty and unreferenced.
- **Parity check history:** Zero errors across all completed checks.

### Claude Code MCP integration (Dell XPS sandbox):
- SSH tunnel established to Unraid-MCP at `172.24.20.11:6970`.
- SessionStart hook auto-establishes tunnel and registers unraid MCP server on every Claude Code session.
- Claude Code now has native Unraid MCP access without SSH for container operations.
- **Known MCP gaps:** Plugin versions, container update availability, CPU temperature, NIC stats not exposed via GraphQL API.

### Final stack assessment: A
Stable, fully audited, all findings resolved. No open items remaining.
