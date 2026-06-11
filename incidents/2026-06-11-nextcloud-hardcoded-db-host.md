# Incident: Nextcloud 500 — Hardcoded DB Host After Container Recreation

Date: 2026-06-11  
Duration: ~2 hours (undetected overnight)  
Severity: Critical  
Status: Resolved  

## Summary
Nextcloud returned HTTP 500 and "Malformed JSON" on mobile after the MariaDB container was recreated, because config.php had a hardcoded IP (192.168.x.x) for the database host that no longer matched the container's new Docker network IP.

## Timeline
- 23:14 — Mobile app shows "Malformed JSON" on sign-in
- 23:20 — SWAG and container logs show repeated SQLSTATE[HY000] [2002] Connection refused errors
- 23:31 — Root cause identified: dbhost hardcoded to stale IP
- 23:33 — config.php updated to use container DNS (mariadb:3306)
- 23:35 — Nextcloud restored, zero data loss

## Root Cause
config.php used a hardcoded Docker bridge IP instead of container DNS. When MariaDB was recreated during a Redis password rotation, Docker assigned it a new IP, breaking the connection string.

## Resolution
Updated dbhost from hardcoded IP to container name:
'dbhost' => 'mariadb:3306'
Docker's embedded DNS resolves container names reliably regardless of IP reassignment.

## Prevention
- Never use hardcoded IPs in application config files — always use container DNS names
- Added to pre-maintenance checklist: verify all config files use container DNS before recreating dependencies

## Related
- Discovered during broader credential rotation audit (2026-06-11)
- MariaDB upgrade banner investigated and confirmed false alarm (already at 11.4.12)
