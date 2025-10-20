---
title: "تست Kafka در Integration Test برای Worker Service"
categories:
  - Net
tags:
  - test
  - kafka
  - mock
---

توسط کتابخانه Testcontainers به راحتی می‌توانید یک نسخه از بیشتر موارد مثل Redis, Sql, Kafka, RabbitMQ را بالا بیاورید و پروژه نوشته شده را تست کنید تا از کارکرد درست آن مطمئن شوید. در این بخش ما Kafka را توسط این روش تست می‌کنیم.  
ابتدا بصورت زیر Zookeeper و سپس Kafka را برای تست‌های خود کانفیگ می‌کینم.  

```c#
using System.Net;
using System.Net.Sockets;
using AutoFixture;
using Confluent.Kafka;
using Confluent.Kafka.Admin;
using DotNet.Testcontainers.Builders;
using Nito.AsyncEx;

namespace IntegrationTest.Setup;

public class TestContainersSetup : ICustomization
{
    private readonly KafkaTestConfig _testContainerConfig;

    public TestContainersSetup()
    {
        _testContainerConfig = TestHelper.LoadTKafkaConfiguration();
    }

    public void Customize(IFixture fixture)
    {
        var zookeeperContainerName = $"zookeeper_{fixture.Create<string>()}";
        var zookeeperContainer = new ContainerBuilder()
            .WithImage("confluentinc/cp-zookeeper:7.6.1")
            .WithName(zookeeperContainerName)
            .WithPortBinding(2181, true)
            .WithEnvironment(new Dictionary<string, string>
            {
                { "ZOOKEEPER_CLIENT_PORT", "2181" },
                { "ZOOKEEPER_TICK_TIME", "2000" }
            })
            .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(2181))
            .Build();

        AsyncContext.Run(async () => await zookeeperContainer.StartAsync());

        var zookeeperHostPort = zookeeperContainer.GetMappedPublicPort(2181);

        var kafkaHostPort = GetAvailablePort();
        var kafkaContainerName = $"kafka_{fixture.Create<string>()}";
        var kafkaContainer = new ContainerBuilder()
            .WithImage("confluentinc/cp-kafka:7.6.1")
            .WithName(kafkaContainerName)
            .WithHostname(kafkaContainerName)
            .WithPortBinding(kafkaHostPort, 9092)
            .WithEnvironment(new Dictionary<string, string>
            {
                { "KAFKA_BROKER_ID", "1" },
                { "KAFKA_ZOOKEEPER_CONNECT", $"host.docker.internal:{zookeeperHostPort}" },
                { "KAFKA_LISTENER_SECURITY_PROTOCOL_MAP", "PLAINTEXT:PLAINTEXT,PLAINTEXT_INTERNAL:PLAINTEXT" },
                { "KAFKA_LISTENERS", "PLAINTEXT://:9092,PLAINTEXT_INTERNAL://:29092" },
                { "KAFKA_ADVERTISED_LISTENERS", $"PLAINTEXT://localhost:{kafkaHostPort},PLAINTEXT_INTERNAL://{kafkaContainerName}:29092" },
                { "KAFKA_INTER_BROKER_LISTENER_NAME", "PLAINTEXT_INTERNAL" },
                { "KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR", "1" },
                { "KAFKA_TRANSACTION_STATE_LOG_MIN_ISR", "1" },
                { "KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR", "1" }
            })
            .WithWaitStrategy(Wait.ForUnixContainer().UntilPortIsAvailable(9092))
            .Build();

        AsyncContext.Run(async () => await kafkaContainer.StartAsync());

        var bootstrapServers = $"{_testContainerConfig.BootstrapServers}:{kafkaHostPort}";

        AsyncContext.Run(async () => await CreateKafkaTopic(_testContainerConfig.TopicName, bootstrapServers));

        fixture.Inject(new KafkaTestConfig
        {
            TopicName = _testContainerConfig.TopicName,
            BootstrapServers = bootstrapServers
        });
    }

    private static int GetAvailablePort()
    {
        IPEndPoint defaultLoopbackEndpoint = new(IPAddress.Loopback, 0);

        using var socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        socket.Bind(defaultLoopbackEndpoint);
        var port = ((IPEndPoint)socket.LocalEndPoint!).Port;

        return port;
    }

    private static async Task CreateKafkaTopic(string topicName, string bootstrapServers)
    {
        using var adminClient = new AdminClientBuilder(new AdminClientConfig { BootstrapServers = bootstrapServers }).Build();

        await adminClient.CreateTopicsAsync(new TopicSpecification[]
        {
            new() { Name = topicName, ReplicationFactor = 1, NumPartitions = 1 }
        });
    }
}
```
سپس بصورت زیر با توجه به نیازمندی می‌توان Consume یا Publish را کانفیگ کرد.  

```c#
using AutoFixture;
using Confluent.Kafka;

namespace IntegrationTest.Setup;

public class KafkaConsumerSetup : ICustomization
{
    public void Customize(IFixture fixture)
    {
        var kafkaConfig = fixture.Create<KafkaTestConfig>();

        var config = new ConsumerConfig
        {
            BootstrapServers = kafkaConfig.BootstrapServers,
            GroupId = Guid.NewGuid().ToString(),
            AutoOffsetReset = AutoOffsetReset.Earliest
        };

        var consumer = new ConsumerBuilder<Null, string>(config).Build();

        consumer.Subscribe(kafkaConfig.TopicName);

        fixture.Inject(consumer);
    }
}
```

```c#
using AutoFixture;
using Confluent.Kafka;

namespace IntegrationTest.Setup;

public class KafkaPublisherSetup : ICustomization
{
    public void Customize(IFixture fixture)
    {
        var kafkaConfig = fixture.Create<KafkaTestConfig>();

        var config = new ConsumerConfig
        {
            BootstrapServers = kafkaConfig.BootstrapServers,
            GroupId = Guid.NewGuid().ToString(),
            AutoOffsetReset = AutoOffsetReset.Earliest
        };

        var producer = new ProducerBuilder<Null, string>(config).Build();

        fixture.Inject(producer);
    }
}
```
در انتها کلاس‌های بالا را بصورت زیر Setup می‌کنیم تا در تست‌های خود از آن استفاده کنیم.  

```c#
using AutoFixture.Xunit2;

namespace IntegrationTest.Setup;

public class DataSetup: AutoDataAttribute
{
    public DataSetup() : base(() => new AutoFixture.Fixture()
        .Customize(new TestContainersSetup())
        .Customize(new KafkaPublisherSetup()))
    {
    }
}
```
در صورتی که برای پروژه نیاز به دیتابیس بود نیز می‌توان بصورت زیر آن را کانفیگ کرد.  
این کانفیگ بعد از ایجاد Image دیتابیس، ساختار دیتابیس را با استفاده از Dacpac که خروجی پروژه sqlproj است می‌سازد و یا از دیتابیس لوکال استفاده می‌کند.  
همچنین توسط Respawn نیز دیتابیس را می‌توانید بعد از هر تست از دیتا خالی کنید.  

```csharp
namespace IntegrationTest.Fixture;

[CollectionDefinition("DatabaseCollection")]
public class DatabaseCollectionFixture : ICollectionFixture<DatabaseFixture>
{
}
```

```csharp
using DotNet.Testcontainers.Builders;
using Microsoft.SqlServer.Dac;
using Respawn;
using Testcontainers.MsSql;

namespace IntegrationTest.Fixture;

public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _msSqlContainer;
    private readonly DatabaseOperation _databaseOperation;
    private readonly TestContainerConfig _testContainerConfig;

    public DatabaseFixture()
    {
        _testContainerConfig = TestHelper.LoadTestContainerConfiguration();

        _msSqlContainer = ConfigureSqlContainer();
        _databaseOperation = new DatabaseOperation(_msSqlContainer, _testContainerConfig);
    }

    public IDbConnector<MyDbContext> DbContext
    {
        get
        {
            var dbContext = new Mock<IDbConnector<MyDbContext>>();
            dbContext.Setup(dc => dc.CreateConnection()).Returns(new SqlConnection(_databaseOperation.ConnectionString));
            return dbContext.Object;
        }
    }

    public string ConnectionString { get; private set; }

    private MsSqlContainer ConfigureSqlContainer()
    {
        var container = new MsSqlBuilder()
            .WithImage(_testContainerConfig.SqlImageAddress)
            .WithPortBinding(_testContainerConfig.SqlPort, _testContainerConfig.SqlContainerPort)
            .WithDockerEndpoint(_testContainerConfig.DockerUri)
            .WithPassword(_testContainerConfig.DatabasePassword)
            .WithAutoRemove(_testContainerConfig.AutoRemoveContainer)
            .WithCleanUp(_testContainerConfig.CleanUp)
            .Build();

        return container;
    }

    private void PublishDatabase()
    {
        var service = new DacServices(ConnectionString + ";TrustServerCertificate=True");
        var dac = DacPackage.Load(GetDacPacPath());

        var dacDeployOptions = new DacDeployOptions();
        dacDeployOptions.SqlCommandVariableValues.Add("DefaultDataPath", "/var/opt/mssql/data/");
        dacDeployOptions.SqlCommandVariableValues.Add("DefaultLogPath", "/var/opt/mssql/log/");

        service.Deploy(dac, "System", false, dacDeployOptions);
    }

    private string GetDacPacPath()
    {
        var path = new DirectoryInfo(CommonDirectoryPath.GetSolutionDirectory().DirectoryPath).Parent?.Parent?.ToString();

        return Path.Combine(path, @"_DataBases\Database\bin\Debug\Database.dacpac");
    }

    public async Task InitializeAsync()
    {
        if (_testContainerConfig.IsLocalDb)
        {
            ConnectionString = _testContainerConfig.LocalConnectionString;
        }
        else
        {
            await _msSqlContainer.StartAsync();
            
            ConnectionString = _databaseOperation.ConnectionString;

            if (_testContainerConfig.FromDbProject)
            {
                PublishDatabase();
            }
            else
            {
                await _databaseOperation.DoOperationAsync(_testContainerConfig.DatabaseName);
            }
        }
    }

    public async Task BlankDatabaseAsync()
    {
        var respawn = await Respawner.CreateAsync(ConnectionString!, new RespawnerOptions
        {
            DbAdapter = DbAdapter.SqlServer,
        });

        await respawn.ResetAsync(ConnectionString);
    }

    public async Task DisposeAsync()
    {
        await BlankDatabaseAsync();
        await _msSqlContainer.DisposeAsync();
    }
}
```

برای اینکه پروژه اصلی شما که می‌تواند یک Worker Service باشد نیز در زمان تست‌ها Host شود نیز از کد زیر می‌توانید استفاده کنید.  


```c#
using AutoFixture;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;

namespace IntegrationTest.Setup;

public class CustomWebApplicationFactory<TStartup> : WebApplicationFactory<TStartup> where TStartup : class
{
    private readonly IFixture fixture;

    private readonly Action<IWebHostBuilder> _configuration;

    public CustomWebApplicationFactory(IFixture fixture) => this.fixture = fixture;

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            var kafkaTestConfig = fixture.Create<KafkaTestConfig>();
            services.Configure<KafkaConfig>(opts =>
            {
                opts.TopicName = kafkaTestConfig.TopicName;
                opts.BootstrapServers = kafkaTestConfig.BootstrapServers;
            });
        });
    }
}
```

در انتها تست شما بصورت زیر است که DatabaseCollection برای دیتابیس، DataSetup برای Kafka استفاده شده است.  


```csharp
using Confluent.Kafka;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Options;
using System.Text.Json;
using FluentAssertions;
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Logging;
using Moq;

namespace IntegrationTest.Test;

[Collection("DatabaseCollection")]
public class WorkerTest : BaseTest
{
    private readonly DatabaseFixture _databaseContainer;
    private IDataUpdater _realDataUpdater;
    private IDataReader _realDataReader;

    public WorkerTest(DatabaseFixture databaseContainer)
    {
        _databaseContainer = databaseContainer;

        ConfigDataUpdater();
    }

    [Theory]
    [DataSetup]
    public async Task GetItemFromQueue_ShouldProcess(IProducer<Null, string> producer, KafkaTestConfig config)
    {
        //Arrange
        _databaseContainer.BlankDatabaseAsync().Wait();
        
        const long RESULT_CUSTOMER_CODE = 0;
        const int COUNT_DATA_BUILDER = 10;

        ConfigDataReader(config);

        var dataGenerated = new MyDataModelDataBuilder().Build(COUNT_DATA_BUILDER);

        foreach (var item in dataGenerated)
        {
            await InsertDataToTableAsync(_databaseContainer.ConnectionString, SqlHelper.MyDataTableInsertQuery, item);

            await ProduceToQueueAsync(producer, config.TopicName, new QueueModel
            {
                CustomerCode = item.CustomerCode,
                Date = item.CreateDate
            });
        }

        var worker = new Worker(Logger.Object, AppSettings.Object, DateTimeProvider.Object, _realDataUpdater, _realDataReader, Mapper);

        // Act
        worker.Execute(JobExecutionContext.Object).Wait();

        // Assert
        var data = await GetDataFromTableAsync<MyDataModel>(_databaseContainer.ConnectionString, SqlHelper.MyDataSelectQuery);

        data.Count.Should().Be(COUNT_DATA_BUILDER);

        foreach (var item in data)
        {
            item.CustomerCode.Should().Be(RESULT_CUSTOMER_CODE);
        }

        _databaseContainer.BlankDatabaseAsync().Wait();
    }

    protected async Task ProduceToQueueAsync(IProducer<Null, string> producer, string topicName, object model)
    {
        await producer.ProduceAsync(topicName, new Message<Null, string>
        {
            Value = JsonSerializer.Serialize(model)
        });
    }
}
```



