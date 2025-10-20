---
title: "استفاده از Docker Image برای تست‌ها"
categories:
  - Net
tags:
  - net
  - test
  - docker
  - container
  - Testcontainers
---

یکی از کتابخانه‌های مناسب برای نوشتن Integration Test در Net core کتابخانه Testcontainers است.  
توسط این کتابخانه می‌توانید به راحتی از سرویس استفاده کننده مانند Sql Server, Redis یک Docker Image مشخص کنید تا یک Container جدید از آن ساخته شود تا بتوانید تست‌ها را در یک محیط ایزوله ران کنید.  
بدین منظور می‌توانید از کد زیر برای ساخت آن استفاده کنید.  
در ابتدا کد در صورتی که می‌خواهید در Github Action نیز تست‌ها را ران کنید. یک شرط قرار داده شده است تا در این حالت از یک workflow استفاده شود.  

```csharp
public class MyTest : IAsyncLifetime
{
    private MsSqlContainer _sqlContainer = null!;
    private RedisContainer _redisContainer = null!;
    private IConnectionMultiplexer _redisConnection = null!;
    private SqlConnection _sqlConnection = null!;

    public async Task InitializeAsync()
    {
        string sqlConnectionString;
        string redisConnectionString;

        // Check if we're in GitHub Actions or local development
        var isCi = Environment.GetEnvironmentVariable("CI") == "true";

        if (isCi)
        {
            // GitHub Actions: Use Docker containers defined in the workflow
            sqlConnectionString = StaticData.SqlConnectionOnRunCi;
            redisConnectionString = StaticData.RedisConnectionOnRunCi;
        }
        else
        {
            var sqlTask = Task.Run(async () =>
            {
                _sqlContainer = new MsSqlBuilder()
                    .WithImage(StaticData.SqlImage)
                    .WithDockerEndpoint(StaticData.DockerEndPoint)
                    .WithPassword(StaticData.DbPassword)
                    .WithAutoRemove(true)
                    .WithCleanUp(true)
                    .WithEnvironment("ACCEPT_EULA", "Y")
                    .WithEnvironment("SA_PASSWORD", StaticData.DbPassword)
                    .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(StaticData.SqlPort))
                    .Build();

                await _sqlContainer.StartAsync();
            });

            var redisTask = Task.Run(async () =>
            {
                _redisContainer = new RedisBuilder()
                    .WithImage(StaticData.RedisImage)
                    .WithDockerEndpoint(StaticData.DockerEndPoint)
                    .WithAutoRemove(true)
                    .WithCleanUp(true)
                    .WithPortBinding(StaticData.RedisPort, true)
                    .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(StaticData.RedisPort))
                    .Build();

                await _redisContainer.StartAsync();
            });

            await Task.WhenAll(sqlTask, redisTask);

            sqlConnectionString = _sqlContainer.GetConnectionString();
            redisConnectionString = _redisContainer.GetConnectionString();
        }

        // Initialize Redis Connection
        _redisConnection = await ConnectionMultiplexer.ConnectAsync(redisConnectionString);

        // Initialize SQL Connection
        _sqlConnection = new SqlConnection(sqlConnectionString);
        await _sqlConnection.OpenAsync();

        // Ensure database setup
        await _sqlConnection.ExecuteAsync(StaticData.QueryToCreateTable);
    }

    public async Task DisposeAsync()
    {
        if (_sqlContainer is not null)
        {
            _sqlConnection.Dispose();

            await _sqlContainer.StopAsync();
            await _sqlContainer.DisposeAsync();
        }

        if (_redisContainer is not null)
        {
            _redisConnection.Dispose();

            await _redisContainer.StopAsync();
            await _redisContainer.DisposeAsync();
        }
    }
}
```

Workflow:  

```yml
name: .NET Core CI with Docker

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    services:
      sql-server:
        image: mcr.microsoft.com/mssql/server:2022-CU13-ubuntu-22.04
        ports:
          - 1433:1433
        options: >-
          --env ACCEPT_EULA=Y
          --env SA_PASSWORD=YourStrong!Passw0rd
          --health-cmd="/opt/mssql-tools/bin/sqlcmd -U sa -P YourStrong!Passw0rd -Q 'SELECT 1'"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=10 
      redis:
        image: redis:latest
        ports:
          - 6379:6379
        options: --health-cmd "redis-cli ping || exit 1"

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up .NET Core
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: '8.0.x'

      - name: Restore dependencies
        run: dotnet restore

      - name: Build the project
        run: dotnet build --configuration Release

      - name: Run tests
        run: dotnet test --configuration Release --no-build
```

اطلاعات بیشتر

[testcontainers](https://dotnet.testcontainers.org/)  
