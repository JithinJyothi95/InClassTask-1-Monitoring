# Branch: fix-query-service-env

## Hypothesis

The `query-service` container existed but returned HTTP 500 errors on key API routes such as `/api/v1/services` and `/api/v1/event`. This suggested a misconfiguration in the internal environment — specifically its connection to ClickHouse and Alertmanager.

## What Was Attempted

After confirming the container was running and healthy, logs revealed internal service errors pointing to missing DSN and alertmanager links. The service was booting, but not connecting to data sources properly.

To fix this, we updated the environment configuration inside `query-service`:

```yaml
environment:
  - ClickHouseUrl=tcp://clickhouse:9000
  - ALERTMANAGER_API_PREFIX=http://alertmanager:9093
```

These were placed correctly under the `query-service` block in `docker-compose-core.yaml`.

## Problem Encountered

Before this fix, the frontend UI could load, but all service and trace data APIs returned 500 errors. This was confirmed through browser devtools and container logs.

## Resolution Steps

1. Updated `query-service` to include required environment variables.
2. Rebuilt and force-recreated the Docker Compose stack:

```bash
docker compose -f clickhouse-setup/docker-compose-core.yaml up -d --force-recreate
```

3. Cleared browser cache to confirm 403 was unrelated and was a stale frontend issue.

## Test Result

After the change, the frontend successfully loaded metrics and services from ClickHouse. 500 errors were resolved, and query-service logs stopped showing backend failures.

## Conclusion

Missing critical environment variables caused `query-service` to malfunction. Supplying `ClickHouseUrl` and `ALERTMANAGER_API_PREFIX` restored full application behavior.
