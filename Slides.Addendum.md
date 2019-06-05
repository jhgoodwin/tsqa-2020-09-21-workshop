
# Addendum

---

# Update IntegrationTest.Dockerfile
```Dockerfile
  test:
    build:
      context: ./
      dockerfile: "IntegrationTest.Dockerfile"
    user: "${UID:-0}:${GID:-0}"
    environment:
      - "ConnectionStrings__Postgres=Host=${COMPOSE_PROJECT_NAME:-citus}_master;Username=postgres"
      - USER=${USER:-root}
    volumes:
      - ${COMMON_TESTRESULTSDIRECTORY:-./test-results}:/app/test-results
    depends_on: { worker: { condition: service_healthy } }
```
---