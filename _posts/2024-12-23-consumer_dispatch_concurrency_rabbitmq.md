---
title: "استفاده از چند Thread برای پردازش پیام‌ها در RabbitMQ با ConsumerDispatchConcurrency"
categories:
  - Net
tags:
  - rabbitmq
  - thread
  - async
---

یکی از قابلیت‌های کمتر توجه شده در ربیت `ConsumerDispatchConcurrency` است. توسط این قابلیت می‌توانید بصورت همزمان چند پیام را خوانده و پردازش کنید.  
برای فعال سازی آن کافی است در `ConnectionFactory` مقدار `ConsumerDispatchConcurrency` را وارد کنید. مقدار پیش‌فرض آن 1 است.  
با مقداردهی آن از این به بعد حداکثر به تعداد گفته شده پیام‌ها بصورت همزمان خوانده و پردازش می‌شود.  
البته با این تغییر ممکن است ترتیب پیام‌ها برای پردازش رعایت نشود.  

```csharp
namespace RabbitmqDetector
{
    public abstract class RabbitMqMultiThreadDetector : IQueueDetector
    {
        private readonly SemaphoreSlim _semaphore = new SemaphoreSlim(1, 1);
        private readonly IModel _channel;
        private readonly object _lockObject = new object();
        private readonly RabbitMqDetectorConfig _config;
        private readonly AsyncEventingBasicConsumer _consumer;
        private readonly IConnection _connection;
        private Task _task;

        protected RabbitMqMultiThreadDetector(RabbitMqDetectorConfig config, int threadCount)
        {
            _config = config;

            if (threadCount < config.PrefetchCount)
            {
                throw new ApplicationException("ThreadCount must me lower than PrefetchCount");
            }

            var connectionFactory = new ConnectionFactory
            {
                UserName = _config.Username,
                Password = _config.Password,
                ClientProvidedName = _config.Name,
                AutomaticRecoveryEnabled = _config.AutomaticRecoveryEnabled,
                DispatchConsumersAsync = true,
                ConsumerDispatchConcurrency = threadCount
            };

            var endpoints = ConvertHostNames(_config.HostNames)
                .Select(item => new AmqpTcpEndpoint(item.Name, item.Port))
                .ToList();

            _connection = connectionFactory.CreateConnection(endpoints);
            _channel = _connection.CreateModel();

            _channel.BasicQos(0, _config.PrefetchCount, false);
            _consumer = new AsyncEventingBasicConsumer(_channel);
            _consumer.Received += HandleReceivedInternal;
        }

        protected async Task AckAsync(ulong deliveryTag)
        {
            await _semaphore.WaitAsync();

            try
            {
                _channel.BasicAck(deliveryTag, false);
            }
            finally
            {
                _semaphore.Release();
            }
        }

        protected async Task NackAsync(ulong deliveryTag)
        {
            await _semaphore.WaitAsync();

            try
            {
                _channel.BasicNack(deliveryTag, false);
            }
            finally
            {
                _semaphore.Release();
            }
        }

        private void Run()
        {
            lock (_lockObject)
            {
                _channel.BasicConsume(_config.Queue, false, _consumer);
            }
        }

        protected abstract Task HandleReceived(QueueMessage result);

        private async Task HandleReceivedInternal(object model, BasicDeliverEventArgs result)
        {
            var data = new QueueMessage
            {
                DeliveryTag = result.DeliveryTag,
                Message = Encoding.UTF8.GetString(result.Body.ToArray()),
                RoutingKey = result.RoutingKey
            };

            await HandleReceived(data);
        }

        public virtual void Dispose()
        {
            if (_channel != null && _channel?.IsOpen == true && _consumer != null)
            {
                _channel.BasicCancel(_consumer.ConsumerTags[0]);

                _channel.QueueUnbind(_config.Queue, _config.Exchange, "1", null);
            }

            _channel?.Dispose();
            _connection?.Dispose();
        }

        public virtual void Stop()
        {
            Dispose();
            _task.Wait();
        }

        private static List<RabbitEndpoint> ConvertHostNames(string hostNames)
        {
            var hosts = hostNames.Split(',');
            return hosts.Select(h => new RabbitEndpoint(h)).ToList();
        }
    }
}
```

دقت کنید که مقدار پاس داده شده به `BasicQos` بیشتر از `threadCount` باشد تا پردازش همزمان درست کار کند.  
همچنین در متود `HandleReceived` حتما باید خطاها و همچنین Ack/Nack مدیریت شود و هیچ خطایی به این لایه داده نشود.  
