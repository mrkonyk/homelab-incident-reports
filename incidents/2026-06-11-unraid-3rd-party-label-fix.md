# Incident: Containers Showing "3rd Party" After Network Segmentation Migration

Date: 2026-06-11  
Duration: ~2 hours investigation  
**Severity:** P3 Low  
Status: Resolved  

## Summary
After a 3-phase Docker network segmentation migration, 13 containers began showing as "3rd Party" in the Unraid UI despite having correct XML templates. Root cause was an undocumented Unraid ownership label missing from containers that were created or recreated outside the Unraid UI.

## Timeline
- 09:00 — Network segmentation migration complete (frontend network emptied)
- 09:30 — Unraid Docker tab shows 13 containers as "3rd Party"
- 10:00 — XML templates verified correct, Docker daemon restarted — no change
- 10:30 — Unraid PHP source (DockerClient.php) inspected to find root cause
- 10:45 — Root cause identified: missing net.unraid.docker.managed label
- 11:00 — All 7 affected containers relabeled and confirmed managed

## Root Cause
Unraid's DockerClient.php getAllInfo() method checks for a Docker container label net.unraid.docker.managed=dockerman before consulting any XML template. If the label is absent, Unraid sets template = null and displays "3rd Party" regardless of whether a valid XML template exists on disk.

Containers created or recreated outside the Unraid UI (via docker run, docker-compose, or migration scripts) do not have this label injected automatically — only containers created through the Unraid Docker UI or the CreateDocker.php mechanism receive it.

Note: Editing /var/lib/docker/containers/<id>/config.v2.json directly does NOT work — Docker holds config in memory and overwrites the file on docker start. The container must be recreated with the label in the docker run command.

## Resolution
For each affected container:
1. docker stop <container>
2. docker rm <container>  
3. Recreate with the ownership label added:
`--label net.unraid.docker.managed=dockerman`

## Prevention
Any container created or recreated outside the Unraid UI must include the ownership label.
This applies to:
- Manual docker run commands
- Rebuild scripts (e.g. honcho rebuild.sh — intentionally excluded)
- Migration scripts
- Any automation that creates containers programmatically

## Checklist for Future Container Recreations
- [ ] Include --label net.unraid.docker.managed=dockerman in docker run
- [ ] Primary network matches XML <Network> field
- [ ] Run network-reattach script after recreation to restore secondary networks
- [ ] Verify label: docker inspect <name> | grep docker.managed

## Related
- Part of the June 11 2026 Docker network segmentation migration
- Affects Unraid 7.3.1 — likely applies to all Unraid versions using the dynamix.docker.manager plugin
