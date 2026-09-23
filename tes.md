## Monitoring

| Area | Service | Status |
|------|---------|--------|
| Job-level failures | Glue + MWAA task logs; jobs log a per-table timing/status summary and raise listing failed tables | **[implemented]** |
| Data agreement with source | `etl.zorro_gold_validation_logs` — one row per table per run | **[implemented]** |
| Pipeline progress / lag | `etl.zorro_etl_logs` — `max(processed_at)` per layer per table | **[implemented]** |
| Workflow alerting | MWAA `on_failure_callback`, `sla_miss_callback` | **[planned]** — no callbacks are wired up |
| DMS replication lag | CloudWatch alarms on `CDCLatencySource` / `CDCLatencyTarget` | **[planned]** |
| Iceberg health | `$files` / `$snapshots` metadata tables | **[planned]** |
| Cost | AWS Budgets + resource tagging | **[planned]** |

The gap worth closing first: a gold task that fails is not retried until new source data arrives for that table, because `CHECK_PENDING_QUERY` only compares bronze vs silver watermarks. A low-traffic table can sit stale indefinitely with nothing alerting.
