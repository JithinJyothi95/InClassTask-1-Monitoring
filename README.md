# Branch: fix-query-service-env

## Hypothesis

The SigNoz query-service container was failing to respond to frontend API calls (403 Forbidden), likely due to missing or improperly set environment variables required for connecting to ClickHouse and Alertmanager.

## What Was Attempted

The following environment variables were added to the `query-service` container in `docker-compose-core.yaml`:

```yaml
environment:
  - ClickHouseUrl=tcp://clickhouse:9000
  - ALERTMANAGER_API_PREFIX=http://alertmanager:9093
```

These variables are required by SigNoz to:
- Connect to ClickHouse for trace and metric data.
- Interact with Alertmanager for alert rule evaluation.

The stack was restarted using Docker Compose.

## Problem Encountered

Prior to adding the variables, the query-service was returning 403 Forbidden on essential API routes. It also failed internal health checks, and no services or events were visible in the frontend UI.

## Resolution Steps

1. Added the required environment variables.
2. Ensured the `clickhouse` service was accessible using internal Docker DNS.
3. Restarted the stack with:

```bash
docker compose -f clickhouse-setup/docker-compose-core.yaml up -d --force-recreate
```

## Test Result

The query-service container became healthy and responded to health probes. Frontend UI loaded properly at `localhost:8085`, and API calls such as `/api/v1/services` began working again.

## Conclusion

Missing critical environment variables prevented query-service from working correctly. Adding `ClickHouseUrl` and `ALERTMANAGER_API_PREFIX` resolved the issue, allowing SigNoz to function normally.