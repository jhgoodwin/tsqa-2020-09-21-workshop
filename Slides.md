---
title : docker-compose lab - John Goodwin
theme : "black"
transition: "fade"
highlightTheme: "darkula"
logoImg: "docker-compose-logo.png"
slideNumber: true
---

# docker-compose testing

2020-09-24 TSQA Hands-on Lab

- John Goodwin
- john@jjgoodwin.com

---

# Agenda

- Reminder - Download Resources
- Describe Lab Goals
- Motivations
- Create new ASP .NET Project
- Intro to Postgres
- Add test project
- Bind test project
- Add migrations
- Bind migrations

---

# Resources

- Download .NET Core 3.1 SDK
    - https://dotnet.microsoft.com/download
- Install Docker for Desktop
    - https://www.docker.com/products/docker-desktop
- Pull these images

    ```bash
    docker pull mcr.microsoft.com/dotnet/core/sdk:3.1
    docker pull mcr.microsoft.com/dotnet/core/aspnet:3.1
    docker pull postgres:12.4
    ```

---

# Resources - Continued

- Docker tutorials
    - https://katacoda.com/
- Postgres
    - https://www.postgresql.org/

---

# Lab Goals

- Emphasis on hands-on for integration testing
- Sideline esoteric discussions
- Postgres Intro
- Emphasis on project/work patterns over design patterns

---

# Motivations

- Technology is hard
- Lots to learn
- Manage risks

---

# Create new ASP .NET Project

```bash
dotnet new sln --name TsqaApi
dotnet new webapi --name TsqaApi
dotnet sln add TsqaApi
```

---

# Intro to Postgres

<https://www.postgresql.org/about/>

<https://www.postgresql.org/about/featurematrix/>

---

# Create Postgres DB

```bash
curl --output docker-compose.yml \
  -O https://raw.githubusercontent.com/docker-library/docs/9efeec18b6b2ed232cf0fbd3914b6211e16e242c/postgres/stack.yml
sed -i 's/image: postgres/image: postgres:12.4/g' \
  docker-compose.yml
docker-compose up
```

---

# Connect

```bash
docker-compose exec db psql -U postgres
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
dotnet new xunit --name TsqaApi.Tests
dotnet sln add TsqaApi.Tests
dotnet test
```

---

# Bind Test Project

IntegrationTest.Dockerfile
```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1
WORKDIR /app
COPY . .
RUN chmod +x run-tests.sh

ENV ConnectionStrings__Postgres ""

CMD ["./run-tests.sh"]
```

---

# run-tests.sh

```sh
#!/usr/bin/env bash
dotnet test
exit $?
```
```sh
chmod +x run-tests.sh
```

---

# Teardown cluster

```bash
docker-compose down
```

---

# Add to docker-compose.yml

```yaml
  test:
    build:
      context: ./
      dockerfile: "IntegrationTest.Dockerfile"
    depends_on: 
      - db
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
dotnet new classlib --name TsqaApi.Migrations
dotnet sln add TsqaApi.Migrations
dotnet add TsqaApi.Migrations package FluentMigrator
```

---

# Add TsqaApi.Migrations/MigrationInitial.cs

```csharp
using FluentMigrator;

namespace TsqaApi.Migrations
{
    /// <summary>
    /// This migration is just to prove that the migrations *CAN* work.
    /// </summary>
    [Migration(1)]
    public class MigrationInitial: Migration
    {
        public override void Up()
        {
            Create.Table("tsqa_temp_table")
                .WithColumn("distribution_id").AsCustom("serial").PrimaryKey()
                .WithColumn("text").AsString().NotNullable();
            Execute.Sql("SELECT * from tsqa_temp_table");
            Delete.Table("tsqa_temp_table");
        }
        public override void Down()
        {
            throw new System.NotImplementedException($"Down not implemented by design");
        }
    }
}
```

---

# Update docker-compose.yml

```yaml
  test:
    build:
      context: ./
      dockerfile: "IntegrationTest.Dockerfile"
    depends_on:
      - db
    environment:
      - "ConnectionStrings__Postgres=Host=db;Username=postgres;Password=example"
```

---

# Update IntegrationTest.Dockerfile

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.1
WORKDIR /app
ENV ConnectionStrings__Postgres ""
ENV MigratorDll "/app/TsqaApi.Migrations/bin/Debug/netstandard2.0/TsqaApi.Migrations.dll"
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
exit $?
```

---

# Restart and run

```bash
docker-compose down; \
docker-compose up --build --exit-code-from test
```

---

# Workshop errors

```bash
test_1     | Time Elapsed 00:00:01.56
test_1     | It was not possible to find any compatible framework version
test_1     | The framework 'Microsoft.NETCore.App', version '2.1.0' was not found.
test_1     |   - The following frameworks were found:
test_1     |       3.1.8 at [/usr/share/dotnet/shared/Microsoft.NETCore.App]
test_1     | 
test_1     | You can resolve the problem by installing the specified framework and/or SDK.
test_1     | 
test_1     | The specified framework can be found at:
test_1     |   - https://aka.ms/dotnet-core-applaunch?framework=Microsoft.NETCore.App&framework_version=2.1.0&arch=x64&rid=debian.10-x64
tsqa-workshop_test_1 exited with code 150
Aborting on container exit...
Stopping tsqa-workshop_adminer_1 ... done
Stopping tsqa-workshop_db_1      ... done
```

---

# It worked!

Script caught errors, but what is wrong?

---

# FluentMigrator SDK 2.1 dependency

IntegrationTest.Dockerfile (insert above tool install)
```Dockerfile
RUN wget https://packages.microsoft.com/config/debian/10/packages-microsoft-prod.deb -O packages-microsoft-prod.deb \
    && dpkg -i packages-microsoft-prod.deb \
    && rm packages-microsoft-prod.deb \
    && apt-get update \
    && apt-get install -y apt-transport-https \
    && apt-get update \
    && apt-get install -y dotnet-sdk-2.1 \
    && dotnet --help \
    && rm -rf /var/lib/apt/lists/*/
```

---

# Strategy works

FluentMigrator issues show this is being worked on, but rather than be stuck, we can add a dependency on SDK 2.1 during testing until it's fixed, then remove it later. Success!

---

# Time permitting

- .dockerignore
- test artifacts

---

# Thank you

by John Goodwin

- https://www.metabolon.com/who-we-are/careers