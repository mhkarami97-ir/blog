---
title: "تست RabbitMQ در Integration Test"
categories:
  - Net
tags:
  - net
  - rabbitmq
  - integration
  - test
---

در مطلبی روش تست کافکا در .net را با هم مرور کردیم:  

[تست کافکا در integration test](https://blog.mhkarami97.ir/net/integration_test_kafka_worker_service/)  


در این مطلب روش تست RabbitMq برای Publish, Consume را بررسی می‌کنیم.  

```csharp
using System.Threading.Tasks;
using Microsoft.IdentityModel.Tokens;
using Testcontainers.RabbitMq;
using Xunit;

namespace TestBase.RabbitMq
{
    public class RabbitmqFixture : IAsyncLifetime
    {
        public string Hostname { get; private set; }
        private readonly RabbitMqContainer _rabbitMqContainer;

        public RabbitmqFixture()
        {
            if (TestConfigs.ShouldUseContainer)
            {
                _rabbitMqContainer = ConfigureContainer();
            }
            else
            {
                Hostname = "127.0.0.1:5672";
            }
        }

        public async Task InitializeAsync()
        {
            if (!TestConfigs.ShouldUseContainer) return;
            await _rabbitMqContainer.StartAsync();
            Hostname = GetHostName();
        }

        public async Task DisposeAsync()
        {
            if (!TestConfigs.ShouldUseContainer) return;
            await _rabbitMqContainer.StopAsync();
        }

        private RabbitMqContainer ConfigureContainer()
        {
            return new RabbitMqBuilder()
                .WithImage("rabbitmq:latest")
                .WithPortBinding(5672, true)
                .WithPortBinding(15672, true)
                .WithEnvironment("RABBITMQ_DEFAULT_USER", "guest")
                .WithEnvironment("RABBITMQ_DEFAULT_PASS", "guest")
                .Build();
        }

        private string GetHostName()
        {
            if (_rabbitMqContainer.Hostname.IsNullOrEmpty()) return string.Empty;

            var url = _rabbitMqContainer.Hostname;
            var port = _rabbitMqContainer.GetMappedPublicPort(5672);
            return url + ':' + port;
        }
    }
}
```

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

namespace TestBase.RabbitMq
{
    public class RabbitMQTestConsumer : IDisposable
    {
        private readonly IConnection _connection;
        private readonly IModel _channel;
        private readonly string _queueName;
        private readonly ConcurrentDictionary<string, TestMessage> _messages;
        private bool _isConsuming;
        private string _consumerTag;

        public RabbitMQTestConsumer(string queueName, string hostname = "localhost:5672")
        {
            _queueName = queueName;
            _messages = new ConcurrentDictionary<string, TestMessage>();

            var factory = new ConnectionFactory { Endpoint = GetEndPoint(hostname) };
            _connection = factory.CreateConnection();
            _channel = _connection.CreateModel();
        }

        public void StartConsuming()
        {
            if (_isConsuming) return;

            var consumer = new EventingBasicConsumer(_channel);
            consumer.Received += HandleReceived;

            _consumerTag = _channel.BasicConsume(
                queue: _queueName,
                autoAck: false,
                consumer: consumer);

            _isConsuming = true;
        }

        private void HandleReceived(object model, BasicDeliverEventArgs ea)
        {
            var body = Encoding.UTF8.GetString(ea.Body.ToArray());
            var messageId = ea.BasicProperties.MessageId ?? Guid.NewGuid().ToString();

            var message = new TestMessage
            {
                MessageId = messageId,
                Headers = ea.BasicProperties.Headers?.ToDictionary(h => h.Key, h => h.Value)
                          ?? new Dictionary<string, object>(),
                Body = body,
                RoutingKey = ea.RoutingKey,
                ReceivedAt = DateTime.UtcNow
            };

            _messages.TryAdd(messageId, message);

            _channel.BasicAck(ea.DeliveryTag, false);
        }

        public void StopConsuming()
        {
            _isConsuming = false;
            _channel.BasicCancel(_consumerTag);
        }

        public IEnumerable<TestMessage> GetAllMessages()
        {
            return _messages.Values.ToList();
        }

        public void ClearMessages()
        {
            _messages.Clear();
            _channel.QueuePurge(_queueName);
        }

        private AmqpTcpEndpoint GetEndPoint(string hostname)
        {
            var host = hostname.Split(':');
            return new AmqpTcpEndpoint(host[0], int.Parse(host[1]));
        }

        public void Dispose()
        {
            _channel?.Close();
            _channel?.Dispose();
            _connection?.Close();
            _connection?.Dispose();
        }
    }
}
```

```csharp
using System;
using System.Threading;
using RabbitMQ.Client;

namespace TestBase.RabbitMq
{
    public class RabbitMqTestProducer : IDisposable
    {
        private readonly IModel _channel;
        private bool _isMutexAcquired;
        private static readonly Mutex PublishMutex = new Mutex();

        public RabbitMqTestProducer(IModel channel)
        {
            _channel = channel;
        }

        public void PublishWithLock(string exchange, string routingKey, byte[] messageBody, IBasicProperties properties = null)
        {
            while (_isMutexAcquired)
            {
                Thread.Sleep(100);
            }

            _isMutexAcquired = PublishMutex.WaitOne();
            _channel.BasicPublish(exchange, routingKey, false, properties, messageBody);
        }

        public void Publish(string exchange, string routingKey, byte[] messageBody, IBasicProperties properties = null)
        {
            _channel.BasicPublish(exchange, routingKey, false, properties, messageBody);
        }

        public void ReleaseLock()
        {
            if (!_isMutexAcquired) return;
            PublishMutex.ReleaseMutex();
            _isMutexAcquired = false;
        }

        public void Dispose()
        {
            _channel?.Dispose();
        }
    }
}
```

```csharp
using System;
using System.Collections.Generic;

namespace TestBase.RabbitMq
{
    public class TestMessage
    {
        public string MessageId { get; set; }
        public Dictionary<string, object> Headers { get; set; }
        public string Body { get; set; }
        public string RoutingKey { get; set; }
        public DateTime ReceivedAt { get; set; }
    }
}
```

```csharp
namespace TestBase.RabbitMq
{
    public class QueueInfo
    {
        public bool Exists { get; set; }
        public uint MessageCount { get; set; }
        public uint ConsumerCount { get; set; }
    }
}
```

```csharp
namespace RabbitmqDetector
{
    public class RabbitMqDetectorConfig
    {
        public string HostNames { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }
        public string Queue { get; set; }
        public string Exchange { get; set; }
        public bool AutomaticRecoveryEnabled { get; set; }
        public ushort PrefetchCount { get; set; }
        public string Name { get; set; }

        public List<AmqpTcpEndpoint> GetAmqpTcpEndpoint()
        {
            return ConvertHostNames(HostNames)
                .Select(item => new AmqpTcpEndpoint(item.Name, item.Port))
                .ToList();
        }

        private static List<RabbitEndpoint> ConvertHostNames(string hostNames)
        {
            var hosts = hostNames.Split(',');
            return hosts.Select(h => new RabbitEndpoint(h)).ToList();
        }
    }
}
```

نمونه تست:  

```csharp
namespace IntegrationTest
{
    [Collection("Test")]
    public class Test : IDisposable
    {
        private readonly IModel _channel;
        private readonly RabbitMqDetectorConfig _detectorConfig;
        private readonly RabbitMQTestConsumer _queueConsumer;
        private readonly RabbitMqTestProducer _rabbitMqTestProducer;

        public Test(MyFixture myFixture)
        {
            var iocContainer = IocContainer.Instance;
            _detectorConfig = iocContainer.Resolve<RabbitMqDetectorConfig>();

            _channel = CreateChannel(_detectorConfig);

            _queueConsumer = new RabbitMQTestConsumer("DispatcherQueue", myFixture.Hostname);
            _rabbitMqTestProducer = new RabbitMqTestProducer(_channel);
        }

        [Fact]
        public void TestQueue()
        {
            // Arrange
            var byteMessage = SerializeMessage("message");
            var properties = CreateMessageHeaders("messageDto");

            _queueConsumer.StartConsuming();

            //Act
            _rabbitMqTestProducer.PublishWithLock("TestExchange", "11", byteMessage, properties);
            Thread.Sleep(2000);

            _queueConsumer.StopConsuming();

            //Assert
            var messages = _queueConsumer.GetAllMessages();
            messages.Should().NotBeEmpty();
        }

        #region Private

        private static IModel CreateChannel(RabbitMqDetectorConfig config)
        {
            var connectionFactory = new ConnectionFactory
            {
                UserName = config.Username,
                Password = config.Password,
                ClientProvidedName = config.Name,
                AutomaticRecoveryEnabled = config.AutomaticRecoveryEnabled
            };
            var connection = connectionFactory.CreateConnection(config.GetAmqpTcpEndpoint());
            return connection.CreateModel();
        }

        private static byte[] SerializeMessage(string message)
        {
            var jsonMessage = JsonSerializer.Serialize(message);
            return Encoding.UTF8.GetBytes(jsonMessage);
        }

        private IBasicProperties CreateMessageHeaders(string messageType)
        {
            var properties = _channel.CreateBasicProperties();
            return properties;
        }

        #endregion

        public void Dispose()
        {
            _channel?.Dispose();
            _queueConsumer?.Dispose();
            _rabbitMqTestProducer?.ReleaseLock();
            _rabbitMqTestProducer?.Dispose();
        }
    }
}
```
