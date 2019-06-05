
# docker-compose testing

2018-06-05 TRINUG Hands-on Lab

- John Goodwin
- john@jjgoodwin.com

---

# Agenda

- Reminder - Download Resources
- Describe Lab Goals
- Motivations
- Create new ASP .NET Project
- Intro to Citus
- Create Citus Cluster
- Add test project
- Bind test project
- Add migrations
- Bind migrations

---

# Resources

- Download .NET Core 2.2.3 SDK
    - https://dotnet.microsoft.com/download
- Install Docker for Desktop
    - https://www.docker.com/products/docker-desktop
- Pull these images

    ```bash
    docker pull mcr.microsoft.com/dotnet/core/sdk:2.2
    docker pull mcr.microsoft.com/dotnet/core/aspnet:2.2
    docker pull citusdata/citus:8.2.1
    docker pull citusdata/membership-manager:0.2.0
    ```

---

# Resources - Continued

- Docker tutorials
    - https://katacoda.com/
- Citus Docker Repo
    - https://github.com/citusdata/docker

---

# Lab Goals

- Emphasis on hands-on for integration testing
- Sideline esoteric discussions
- Citus Intro
- Emphasis on project/work patterns over design patterns

---

# Motivations

- Technology is hard
- Lots to learn
- Manage risks

---

# Create new ASP .NET Project

```bash
dotnet new sln --name TrinugApi
dotnet new webapi --name TrinugApi
dotnet sln add TrinugApi
```

---

# Intro to Citus 

https://www.citusdata.com/product

_(click See How)_

---

# Create Citus Cluster

```bash
curl -O https://raw.githubusercontent.com/citusdata/docker/master/docker-compose.yml
docker-compose up
```

---

# Connect

```bash
docker exec -it citus_master psql -U postgres
```
Then:
```sql
select version();

select current_timestamp;

\dt *.*
```

---

# Add Test Project

```
dotnet new xunit --name TrinugApi.Tests
dotnet sln add TrinugApi.Tests
dotnet test
```

---

# Bind Test Project

IntegrationTest.Dockerfile
```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.2
WORKDIR /app
COPY . .

ENV ConnectionStrings__Postgres ""

CMD ["./run-tests.sh"]
```

---

# run-tests.sh

```sh
#!/usr/bin/env bash
dotnet test
```
Then:
```bash
chmod +x run-tests.sh
```

---

# Teardown cluster

```bash
docker-compose down
```

---

# Add to IntegrationTest.Dockerfile

```Dockerfile
  test:
    build:
      context: ./
      dockerfile: "IntegrationTest.Dockerfile"
    depends_on: { worker: { condition: service_healthy } }
```

---

# Restart Cluster

```bash
docker-compose up --exit-code-from test
```

---

# Add Migrations

https://fluentmigrator.github.io/articles/intro.html

```bash
dotnet tool install -g FluentMigrator.DotNet.Cli
```

---

# Add Migrations Project

```bash
dotnet new classlib --name TrinugApi.Migrations
dotnet sln add TrinugApi.Migrations
dotnet add TrinugApi.Migrations package FluentMigrator
```

---

# Add MigrationInitial.cs

```csharp
using FluentMigrator;

namespace TrinugApi.Migrations
{
    /// <summary>
    /// This migration is just to prove that the migrations *CAN* work.
    /// </summary>
    [Migration(1)]
    public class MigrationInitial: Migration
    {
        public override void Up()
        {
            Create.Table("trinug_temp_table")
                .WithColumn("distribution_id").AsCustom("serial").PrimaryKey()
                .WithColumn("text").AsString().NotNullable();
            Execute.Sql("SELECT create_distributed_table('trinug_temp_table', 'distribution_id')");
            Delete.Table("trinug_temp_table");
        }
        public override void Down()
        {
            // this space intentionally left blank.
        }
    }
}
```

---

# Update IntegrationTest.Dockerfile

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:2.2
WORKDIR /app
ENV ConnectionStrings__Postgres ""
ENV MigratorDll "/app/TrinugApi.Migrations/bin/Debug/netstandard2.0/TrinugApi.Migrations.dll"
RUN dotnet tool install -g FluentMigrator.DotNet.Cli
ENV HOME="/root"
ENV PATH="${HOME}/.dotnet/tools:${PATH}"

COPY . .

CMD ["./run-tests.sh"]
```

---

# Update run-tests.sh

```sh
#!/usr/bin/env bash
dotnet restore --packages packages \
    && dotnet build --packages packages \
    && dotnet fm migrate -p postgres \
      -c $ConnectionStrings__Postgres \
      -a $MigratorDll \
    && dotnet test
```

---

# Restart and run

```bash
docker-compose down; \
docker-compose up --build --exit-code-from test
```

---

# Time permitting

- .dockerignore
- test artifacts

---

# Thank you

by John Goodwin

- Food sponsored by Metabolon
- Take pic for my manager!
- https://www.metabolon.com/who-we-are/careers