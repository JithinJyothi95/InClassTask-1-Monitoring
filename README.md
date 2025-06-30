# Branch: fix-add-query-service

## Hypothesis

The SigNoz application failed to start because the `query-service` container was entirely missing from the `docker-compose-core.yaml` setup.

## What Was Attempted

To investigate the startup failure, the stack was reviewed and it was confirmed that the `query-service` (which is essential for bridging ClickHouse data and the frontend UI) was not defined in the active Compose file.

A full configuration for `query-service` was added, including environment variables, ports, healthcheck, and volumes.

### Added service definition:

```yaml
query-service:
  image: signoz/query-service:0.73.0
  ports:
    - "8080:8080"
    - "8085:8080"
  depends_on:
    - clickhouse
  command: ["./query-service", "-config=/root/config.yml", "-dsn=tcp://clickhouse:9000"]
  environment:
    - ClickHouseUrl=tcp://clickhouse:9000
    - ALERTMANAGER_API_PREFIX=http://alertmanager:9093
  volumes:
    - ./data/sqlite:/var/lib/signoz
  healthcheck:
    test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/api/v1/health"]
    interval: 10s
    timeout: 5s
    retries: 5
```

The stack was restarted using Docker Compose.

## Problem Encountered

Before the fix, launching the full stack failed due to unmet dependency conditions on `query-service`. Other services like `otel-collector` and `alertmanager` could not initialize fully. UI returned HTTP 403 errors — which were later confirmed as a browser cache artifact due to failed previous loads.

## Resolution Steps

1. Added the `query-service` block in `docker-compose-core.yaml`.
2. Mapped correct ports (8080 internally and 8085 externally).
3. Ensured environment and healthcheck were in place.
4. Ran the following command:

```bash
docker compose -f clickhouse-setup/docker-compose-core.yaml up -d --force-recreate
```

## Test Result

After applying the fix, all containers started properly. The query-service became healthy, and SigNoz UI at `localhost:8085` began displaying traces, metrics, and service data.

## Conclusion

The root cause of the application failure was a missing `query-service`. Once added and restarted, the SigNoz platform was able to fully boot and serve requests.
