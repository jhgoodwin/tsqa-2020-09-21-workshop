
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
      - "ConnectionStrings__Postgres=Host=db;Username=postgres;Password=example"
      - USER=${USER:-root}
    volumes:
      - ${COMMON_TESTRESULTSDIRECTORY:-./test-results}:/app/test-results
    depends_on: { db: { condition: service_healthy } }
```
---