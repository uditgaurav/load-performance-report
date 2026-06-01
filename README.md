# CODE_API_TEST — Scenario Benchmark Report

**Date:** 2026-06-01  
**Scope:** Per-scenario runs only (`code-qa-v13`). Excludes matrix runner results and the mistaken `v14_500u_3w_300s` run (`code-qa-v14`).

---

## 1. Test setup

| Item | Value |
|------|--------|
| Locust image | `uditgaurav/locust-tasks:code-qa-v13` |
| Scenario | `CODE_API_TEST` (`loadArgs`: `test-scenario=CODE_API_TEST;env=on-prem`) |
| Target URL | `https://udit.pr2.harness.io/gateway` |
| Ramp | 60s |
| Steady load | 300s per scenario |
| Control plane | `gke_hce-play_us-west1_chaos-team-standard`, ns `udit` (LTM, Mongo, k8s-ifs) |
| Load infra | `gke_hce-play_us-east1_udit-load-test-cluster`, ns `load` (DDCR, Locust) |
| Sampling | `kubectl top` every 10s → `top_samples.csv`; Mongo baseline/post + per-`runId` delta |

---

## 2. Scenarios in scope

| Scenario | Users | Workers | Run ID | Archive |
|----------|------:|--------:|--------|---------|
| `v13_100u_2w_300s` | 100 | 2 | `5ca7578b-8a72-4e60-81ce-07197b37af10` | `scenarios/v13_100u_2w_300s_1780352355/` |
| `v13_500u_3w_300s` | 500 | 3 | `e630f985-6173-46d0-b831-723a1f677c44` | `scenarios/v13_500u_3w_300s_1780352648/` |
| `v13_1000u_7w_300s` | 1000 | 7 | `ca998194-401b-4ac4-9f01-d1969a647932` | `scenarios/v13_1000u_7w_300s_1780353112/` |

**Excluded:** `v14_500u_3w_300s` — wrong image tag (`code-qa-v14`).

---

## 3. Resource utilization (`kubectl top`)

CPU in millicores; memory in MiB. Values from `summary.json` (aggregated over capture window).

### 3.1 Control plane & orchestration

| Scenario | Component | CPU avg | CPU max | CPU p95 | Mem avg | Mem max | Samples |
|----------|-----------|--------:|--------:|--------:|--------:|--------:|--------:|
| 100u / 2w | LTM | 7 | 7 | 7 | 36 | 36 | 1 |
| 100u / 2w | k8s-ifs | 5 | 5 | 5 | 25 | 25 | 1 |
| 500u / 3w | LTM | 3.7 | 4 | 4 | 44 | 44 | 3 |
| 500u / 3w | k8s-ifs | 4.7 | 6 | 6 | 26.7 | 27 | 3 |
| 1000u / 7w | LTM | 11.5 | 51 | 51 | 42.8 | 45 | 12 |
| 1000u / 7w | k8s-ifs | 4.7 | 6 | 6 | 25.6 | 27 | 12 |
| 100u / 2w | DDCR | 9 | 9 | 9 | 41 | 41 | 1 |
| 500u / 3w | DDCR | 14.5 | 19 | 19 | 40 | 41 | 2 |
| 1000u / 7w | DDCR | 8.5 | 10 | 10 | 40.5 | 41 | 2 |

LTM stays light at 100–500 users; a short burst to **51m CPU** appears at 1000 users (p95 = max with 12 samples).

### 3.2 Locust (load cluster)

| Scenario | Component | CPU avg | CPU max | CPU p95 | Mem avg | Mem max | Worker pods (end) | Samples |
|----------|-----------|--------:|--------:|--------:|--------:|--------:|------------------:|--------:|
| 100u / 2w | master | 28 | 28 | 28 | 41 | 41 | 1 | 1 |
| 100u / 2w | workers (agg) | 446 | 446 | 446 | 105 | 105 | 1† | 1 |
| 500u / 3w | master | 28 | 32 | 32 | 41.5 | 43 | 1 | 2 |
| 500u / 3w | workers (agg) | 269.8 | 479 | 479 | 92 | 150 | 3 | 6 |
| 1000u / 7w | master | 39 | 42 | 42 | 43 | 44 | 1 | 2 |
| 1000u / 7w | workers (agg) | 216.2 | 416 | 383 | 100.7 | 168 | 7 | 20 |

† **100u caveat:** Archived `summary.json` references run `9235f5c0-…` and a single worker pod from an overlapping run; manifest records the intended run as `5ca7578b-…`. Treat 100u **kubectl top** numbers as **unreliable** (one sample, wrong run mix).

**Locust limits:** All scenarios — `has_limits: false` (no CPU/memory requests or limits on master/worker containers).

---

## 4. MongoDB growth (per run)

| Scenario | Run ID | Request logs (docs) | Request logs (size) | Metrics timeseries (docs) | DB total delta‡ |
|----------|--------|--------------------:|--------------------:|--------------------------:|----------------:|
| 100u / 2w | `5ca7578b-…` | 450 | ~87 KiB | 30 | — |
| 500u / 3w | `e630f985-…` | 8,290 | 1.57 MiB | 51 | 1.49 MiB |
| 1000u / 7w | `ca998194-…` | 16,074 | 3.31 MiB | 71 | — |

‡ From `mongo_delta.json` (`dbTotal_delta_mb`) where capture succeeded.

- **500u:** Full harness capture (`mongo_delta.json`).
- **1000u:** Harness reported `aggregation failed`; sizes above from a **post-hoc** `runId` query (same method as benchmarker).
- **100u:** Archive `mongo_delta` tied to wrong run; sizes above from post-hoc query for `5ca7578b-…`.

Rough scaling: request-log volume grows ~18× from 100u → 500u and ~2× from 500u → 1000u (docs), consistent with higher concurrency and longer stable sampling on later runs.

---

## 5. Data quality & operational notes

| Issue | Affected scenario | Impact |
|-------|-------------------|--------|
| Overlapping / wrong run in archive | `v13_100u_2w_300s` | `summary.json` run_id ≠ manifest; 1 top sample; DDCR/Locust pods from `9235f5c0` | Use manifest run_id + post-hoc Mongo only |
| Sparse DDCR/Locust samples | `v13_100u_2w`, `v13_500u_3w` | 1–3 LTM samples vs 12 at 1000u | Avg/max less representative |
| Mongo post snapshot missing | `v13_1000u_7w` | No `mongo_post.json` in archive | Per-run query used for report |
| Wrong image run | `v14_500u_3w_300s` | Excluded from this report | — |

**Observed elsewhere in this benchmark campaign (not re-measured per row above):** Locust workers may restart once under load; UI user count can drop when workers miss heartbeat; worker logs may show `INVALID_CREDENTIAL` while UI error rate stays 0%; DDCR teardown deletes Locust deployments at run end (normal, not OOM).

---

## 6. Summary

| Scenario | Locust workers | Worker CPU p95 | Worker mem max | Request logs | Notes |
|----------|---------------:|-----------------:|---------------:|-------------:|-------|
| 100u / 2w | 2 (intended) | — | — | 450 docs | Resource metrics **not trustworthy** in archive |
| 500u / 3w | 3 | 479m | 150 MiB | 8.3k docs, ~1.6 MiB | Best balanced capture quality |
| 1000u / 7w | 7 | 383m | 168 MiB | 16.1k docs, ~3.3 MiB | LTM CPU spike to 51m; Mongo delta capture failed in harness |

**Takeaway:** At `code-qa-v13`, control-plane components (LTM, k8s-ifs, DDCR) remain modest; **Locust workers** dominate CPU/memory and scale with user/worker count. Mongo request-log growth is measurable per run and scales with load. For reporting, trust **`summary.json` run_id** and **`mongo_delta.json`** only when they match the manifest; re-query Mongo by `runId` when they do not.

---

## 7. Artifacts

| Path | Contents |
|------|----------|
| `scenarios/manifest.jsonl` | Run metadata, completion status |
| `scenarios/INDEX.md` | Quick index (some run_ids may be stale) |
| `scenarios/<name>_<ts>/test_1_<N>users/` | `summary.json`, `top_samples.csv`, `mongo_*.json`, `locust_limits.json` |
| `scenarios/SCENARIOS_BENCHMARK_REPORT.md` | This report |
