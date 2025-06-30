# Branch: fix-add-query-service

## Hypothesis

The SigNoz application was returning 403 errors and failing to serve metrics and services because the `query-service` container was entirely missing from the deployment stack.

## What Was Attempted

To address this, a new `query-service` container was added to the `docker-compose-core.yaml` under the correct service definition.

Here is the configuration that was added:

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

The stack was then restarted using Docker Compose.

## Problem Encountered

Before this change, accessing the frontend returned HTTP 403 errors for key API routes like `/api/v1/services`, `/api/v1/event`, and `/api/v3/autocomplete/attribute_keys`. Logs showed the requests were hitting the middleware but returned 403 consistently.

## Resolution Steps

1. Added the `query-service` container to `docker-compose-core.yaml`.
2. Included health checks and environment variables.
3. Used the correct image tag and port mapping.
4. Ran the following command to apply the fix:

```bash
docker compose -f clickhouse-setup/docker-compose-core.yaml up -d --force-recreate
```

## Test Result

After the change, all containers came up successfully. The query-service passed its health check, and the frontend UI loaded the dashboards and services correctly via `localhost:8085`. HTTP 403 errors were resolved.

## Conclusion

The absence of the `query-service` was the root cause of failed API access. Adding it properly restored full functionality. This fix ensures SigNoz has a working backend to query ClickHouse and serve API responses to the frontend.