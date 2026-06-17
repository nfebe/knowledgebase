# Outage 2026-06-06: org/project selector empties on team.useblocks.com

## TL;DR

The org/project/version selector intermittently showed no options, then under load failed persistently: `GET /api/v1/artifacts/organizations` hung and never returned (observed 7 called, 0 responses). Root cause is application code, not deployment or infrastructure.

The selector list comes from `UbtSphinxParserService.getAllProjectConfig()`. On a cache miss it rebuilds the list by recursively walking the entire `/data/output` EFS tree for `ubtrace_project.toml` files. Measured live: 27 files across 3,270 directories, 3,299 sequential EFS round trips, 149 seconds. The result is cached in Redis (`ubt_all_project_config`) with a 1-hour TTL, and the rebuild runs synchronously on the request path with no single-flight guard. When the cache is cold, every ~30s frontend retry launches another full walk; on the single-threaded event loop they stack and never finish, so the cache never repopulates.

This is the same EFS-scan cost as the startup hang in [outage-2026-05-25](./outage-2026-05-25.md) / #1418, reached through a different entry point (selector request vs onModuleInit boot).

Aggravator: the worker ingestion loop (#1419/#1423) kept EFS saturated, pushing the selector from "slow/intermittent" to "permanently hung."

## Evidence (all captured live, pre-mitigation)

- Infra healthy throughout: all pods Running, Elasticsearch GREEN with 0 thread-pool rejections, Postgres and Redis up, Redis 90M/384M with `evicted_keys=0`, app reached ES in 2-24 ms. Rules out infra/deployment.
- App-style walk measured in-pod: `found=27 listCalls=3299 wall=149318 ms`.
- API logs: `GET /api/v1/artifacts/organizations` called 7x, 0 `RES status` responses (hang, not crash or 500).
- Redis: `ubt_all_project_config` absent (`EXISTS=0`); keyspace miss rate ~82% (1.41M miss vs 0.31M hit).
- Worker queue `ubtrace:worker:queue` LLEN 246,033; drained ~165 tasks in 10 hours (effectively stuck). 8,000-item sample: 100% `action:load`, only ~18 unique org/project/version, each duplicated ~477x. Scheduler logs every ~30s: `ASPICE config change detected: useblocks/ubconnect (manifest=none, delta=new)` and `Found 9 version(s) with outdated NDJSON format` and `Queue backpressure: 246k`.

## Why the selector is NOT ES-backed (corrects the first hypothesis)

`artifact-organization.service.adapter.ts` and `artifact-project.service.adapter.ts` inject `UbtSphinxParserService`, not Elasticsearch. The readiness flag `search: degraded "Elasticsearch unreachable"` seen during the incident is a side effect: while the single api event loop is stuck in the walk, the ES health `pingFresh` (2000ms) times out. It is not the selector's data source.

## Cache invalidation detail

`worker-processor.service.ts` only deletes `ubt_all_project_config` on a project-set change (`isProjectSetChange=true`); normal re-processing relies on the 1h TTL. So the empty-cache state is TTL expiry plus failed rebuild, not constant deletion.

## Mitigation applied (reversible, in order)

1. Scaled worker to 0 to free EFS. Not enough on its own: accumulated in-flight walks kept the event loop saturated.
2. `kubectl rollout restart deploy/ubtrace-api`. Surge kept the old pod serving; the new pod ran the walk once at boot (quiet EFS, no concurrent retries), wrote the cache in ~100s, became Ready. Selector restored (`organizations` 200 in 15ms).
3. Scaled worker back to 1.
4. Cleared the stuck queue safely via `RENAME ubtrace:worker:queue ubtrace:worker:queue:bak-2026-06-06` (atomic empty + full backup), set 7-day EXPIRE on the backup. Live queue 246,033 to 0; backpressure logs gone; scheduler now finds only 1-2 outdated versions per scan.

After the clear, a cold-cache rebuild completed in ~25s (quiet EFS) instead of hanging, then served 3ms cache hits. Confirmed the fix held.

## What is fixed vs not

- Fixed (operationally): EFS no longer saturated; selector recovers on cold cache instead of hanging.
- Not fixed (code): the 149s synchronous full-tree scan, the 1h TTL, and the missing single-flight lock remain. A cold-cache hit still costs one slow load (~25s on quiet EFS) per hour until the code fix.
- The queue clear is relief, not a cure: the re-enqueue loop (ubconnect `manifest=none`, versions never marked current) is unfixed and will slowly refill (bounded by backpressure 256).

## Related issues

- #1477 selector code fix: background refresh, single-flight lock, drop the full-tree scan (index config locations or persist the project list in Postgres).
- #1423 worker queue duplicate backlog (aggravator); manual unblock performed today.
- #1418 same EFS scan blocks api startup.
- #1385 worker manifest read+written to EFS every task (adds EFS contention).
