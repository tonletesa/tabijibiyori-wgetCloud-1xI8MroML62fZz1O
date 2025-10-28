[合集 - 消息队列RabbitMQ教程(9)](https://github.com)

[1.【RabbitMQ】环境搭建，版本选择和安装，添加用户授权2023-12-31](https://github.com/jixingsuiyuan/p/15905326.html)[2.【RabbitMQ】消息队列理论部分，另一种环境搭建Docker运行RabbitMQ09-13](https://github.com/jixingsuiyuan/p/19089393)[3.【RabbitMQ】核心模型简介，以及消息的生产与消费09-20](https://github.com/jixingsuiyuan/p/19103083)[4.【RabbitMQ】工作队列（Work Queues）与消息确认（Ack）09-21](https://github.com/jixingsuiyuan/p/19104405):[蓝猫加速器官网](https://baoshidao.com)[5.【RabbitMQ】发布/订阅（Publish/Subscribe）与交换机（Exchange）09-22](https://github.com/jixingsuiyuan/p/19106408)[6.【RabbitMQ】路由（Routing）与直连交换机（Direct Exchange）09-24](https://github.com/jixingsuiyuan/p/19108392)[7.【RabbitMQ】主题（Topics）与主题交换机（Topic Exchange）09-26](https://github.com/jixingsuiyuan/p/19114022)[8.【RabbitMQ】实现完整的消息可靠性保障体系09-28](https://github.com/jixingsuiyuan/p/19117336)

9.【RabbitMQ】与ASP.NET Core集成10-28

收起

## 本章目标

* 掌握在ASP.NET Core中配置和依赖注入RabbitMQ服务。
* 学习使用`IHostedService`/`BackgroundService`实现常驻消费者服务。
* 实现基于RabbitMQ的请求-响应模式。
* 构建完整的微服务间异步通信解决方案。
* 学习配置管理和健康检查。

---

## 一、理论部分

### 1. ASP.NET Core集成模式

将RabbitMQ集成到ASP.NET Core应用程序时，我们需要考虑几个关键方面：

* 依赖注入：正确管理连接和通道的生命周期。
* 托管服务：实现后台消息消费者。
* 配置管理：从配置文件读取RabbitMQ连接设置。
* 健康检查：监控RabbitMQ连接状态。
* 日志记录：使用ASP.NET Core的日志系统。

### 2. 生命周期管理

* IConnection：建议注册为单例，因为创建TCP连接开销大。
* IModel：建议注册为瞬态或作用域，因为通道不是线程安全的。
* 生产者服务：可以注册为作用域或瞬态。
* 消费者服务：通常在托管服务中管理。

### 3. 托管服务（Hosted Services）

ASP.NET Core提供了`IHostedService`接口和`BackgroundService`基类，用于实现长时间运行的后台任务。这是实现RabbitMQ消费者的理想方式。

### 4. 微服务架构中的消息模式

* 异步命令：发送指令但不期待立即响应。
* 事件通知：广播状态变化。
* 请求-响应：类似RPC，但通过消息中间件。

---

## 二、实操部分：构建订单处理微服务

我们将创建一个完整的订单处理系统，包含：

* Order.API：接收HTTP订单请求，发布消息
* OrderProcessor.BackgroundService：后台处理订单
* 订单状态查询API
* 健康检查
* 配置管理

### 第1步：创建项目结构

```
# 创建解决方案
dotnet new sln -n OrderSystem

# 创建Web API项目
dotnet new webapi -n Order.API
dotnet new classlib -n Order.Core
dotnet new classlib -n Order.Infrastructure
dotnet new classlib -n OrderProcessor.Service

# 添加到解决方案
dotnet sln add Order.API/Order.API.csproj
dotnet sln add Order.Core/Order.Core.csproj
dotnet sln add Order.Infrastructure/Order.Infrastructure.csproj
dotnet sln add OrderProcessor.Service/OrderProcessor.Service.csproj

# 添加项目引用
dotnet add Order.API reference Order.Core
dotnet add Order.API reference Order.Infrastructure
dotnet add OrderProcessor.Service reference Order.Core
dotnet add OrderProcessor.Service reference Order.Infrastructure
dotnet add Order.Infrastructure reference Order.Core

# 添加NuGet包
cd Order.API
dotnet add package RabbitMQ.Client
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks

cd ../Order.Infrastructure
dotnet add package RabbitMQ.Client
dotnet add package Microsoft.Extensions.Configuration

cd ../OrderProcessor.Service
dotnet add package RabbitMQ.Client
```

### 第2步：定义领域模型（Order.Core）

Models/Order.cs

```
namespace Order.Core.Models
{
    public class Order
    {
        public string Id { get; set; } = Guid.NewGuid().ToString();
        public string CustomerId { get; set; }
        public string ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal TotalAmount { get; set; }
        public OrderStatus Status { get; set; } = OrderStatus.Pending;
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public DateTime? ProcessedAt { get; set; }
    }

    public enum OrderStatus
    {
        Pending,
        Processing,
        Completed,
        Failed,
        Cancelled
    }
}
```

Messages/OrderMessage.cs

```
namespace Order.Core.Messages
{
    public class OrderMessage
    {
        public string OrderId { get; set; }
        public string CustomerId { get; set; }
        public string ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal TotalAmount { get; set; }
        public string Action { get; set; } // "create", "cancel"
    }

    public class OrderStatusMessage
    {
        public string OrderId { get; set; }
        public OrderStatus Status { get; set; }
        public string Message { get; set; }
        public DateTime Timestamp { get; set; } = DateTime.UtcNow;
    }
}
```

### 第3步：基础设施层（Order.Infrastructure）

Services/IRabbitMQConnection.cs

```
using RabbitMQ.Client;

namespace Order.Infrastructure.Services
{
    public interface IRabbitMQConnection : IDisposable
    {
        bool IsConnected { get; }
        IModel CreateModel();
        bool TryConnect();
    }
}
```

Services/RabbitMQConnection.cs

```
using System.Net.Sockets;
using Microsoft.Extensions.Logging;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using RabbitMQ.Client.Exceptions;

namespace Order.Infrastructure.Services
{
    public class RabbitMQConnection : IRabbitMQConnection
    {
        private readonly IConnectionFactory _connectionFactory;
        private readonly ILogger _logger;
        private IConnection _connection;
        private bool _disposed;
        
        private readonly object _syncRoot = new object();

        public RabbitMQConnection(IConnectionFactory connectionFactory, ILogger logger)
        {
            _connectionFactory = connectionFactory;
            _logger = logger;
        }

        public bool IsConnected => _connection != null && _connection.IsOpen && !_disposed;

        public IModel CreateModel()
        {
            if (!IsConnected)
            {
                throw new InvalidOperationException("No RabbitMQ connections are available to perform this action");
            }

            return _connection.CreateModel();
        }

        public bool TryConnect()
        {
            lock (_syncRoot)
            {
                if (IsConnected) return true;

                _logger.LogInformation("RabbitMQ Client is trying to connect");

                try
                {
                    _connection = _connectionFactory.CreateConnection();
                    
                    _connection.ConnectionShutdown += OnConnectionShutdown;
                    _connection.CallbackException += OnCallbackException;
                    _connection.ConnectionBlocked += OnConnectionBlocked;

                    _logger.LogInformation("RabbitMQ Client acquired a persistent connection to '{HostName}' and is subscribed to failure events", 
                        _connectionFactory.HostName);

                    return true;
                }
                catch (BrokerUnreachableException ex)
                {
                    _logger.LogError(ex, "RabbitMQ connection failed: {Message}", ex.Message);
                    return false;
                }
                catch (SocketException ex)
                {
                    _logger.LogError(ex, "RabbitMQ connection failed: {Message}", ex.Message);
                    return false;
                }
            }
        }

        private void OnConnectionBlocked(object sender, ConnectionBlockedEventArgs e)
        {
            if (_disposed) return;

            _logger.LogWarning("A RabbitMQ connection is blocked. Reason: {Reason}", e.Reason);
            
            // 这里可以实现重连逻辑
            TryConnect();
        }

        private void OnCallbackException(object sender, CallbackExceptionEventArgs e)
        {
            if (_disposed) return;

            _logger.LogWarning(e.Exception, "A RabbitMQ connection throw exception. Trying to re-connect...");
            
            // 这里可以实现重连逻辑
            TryConnect();
        }

        private void OnConnectionShutdown(object sender, ShutdownEventArgs reason)
        {
            if (_disposed) return;

            _logger.LogWarning("A RabbitMQ connection is on shutdown. Trying to re-connect...");

            // 这里可以实现重连逻辑
            TryConnect();
        }

        public void Dispose()
        {
            if (_disposed) return;

            _disposed = true;

            try
            {
                _connection?.Dispose();
            }
            catch (IOException ex)
            {
                _logger.LogCritical(ex, "Error disposing RabbitMQ connection");
            }
        }
    }
}
```

Services/IOrderPublisher.cs

```
using Order.Core.Messages;

namespace Order.Infrastructure.Services
{
    public interface IOrderPublisher
    {
        Task PublishOrderCreatedAsync(OrderMessage order);
        Task PublishOrderStatusAsync(OrderStatusMessage status);
    }
}
```

Services/OrderPublisher.cs

```
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Logging;
using Order.Core.Messages;
using RabbitMQ.Client;

namespace Order.Infrastructure.Services
{
    public class OrderPublisher : IOrderPublisher
    {
        private readonly IRabbitMQConnection _connection;
        private readonly ILogger _logger;
        private const string ExchangeName = "order.events";
        private const string OrderCreatedRoutingKey = "order.created";
        private const string OrderStatusRoutingKey = "order.status";

        public OrderPublisher(IRabbitMQConnection connection, ILogger logger)
        {
            _connection = connection;
            _logger = logger;
            
            // 确保交换机和队列存在
            InitializeInfrastructure();
        }

        private void InitializeInfrastructure()
        {
            using var channel = _connection.CreateModel();
            
            // 声明主题交换机
            channel.ExchangeDeclare(ExchangeName, ExchangeType.Topic, durable: true);
            
            // 声明订单创建队列
            channel.QueueDeclare("order.created.queue", durable: true, exclusive: false, autoDelete: false);
            channel.QueueBind("order.created.queue", ExchangeName, OrderCreatedRoutingKey);
            
            // 声明订单状态队列
            channel.QueueDeclare("order.status.queue", durable: true, exclusive: false, autoDelete: false);
            channel.QueueBind("order.status.queue", ExchangeName, OrderStatusRoutingKey);
            
            _logger.LogInformation("RabbitMQ infrastructure initialized");
        }

        public async Task PublishOrderCreatedAsync(OrderMessage order)
        {
            await PublishMessageAsync(order, OrderCreatedRoutingKey, "OrderCreated");
        }

        public async Task PublishOrderStatusAsync(OrderStatusMessage status)
        {
            await PublishMessageAsync(status, OrderStatusRoutingKey, "OrderStatus");
        }

        private async Task PublishMessageAsync(T message, string routingKey, string messageType)
        {
            if (!_connection.IsConnected)
            {
                _connection.TryConnect();
            }

            using var channel = _connection.CreateModel();
            
            var json = JsonSerializer.Serialize(message);
            var body = Encoding.UTF8.GetBytes(json);

            var properties = channel.CreateBasicProperties();
            properties.Persistent = true;
            properties.ContentType = "application/json";
            properties.Type = messageType;

            try
            {
                channel.BasicPublish(
                    exchange: ExchangeName,
                    routingKey: routingKey,
                    mandatory: true,
                    basicProperties: properties,
                    body: body);

                _logger.LogInformation("Published {MessageType} message for Order {OrderId}", 
                    messageType, GetOrderId(message));
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error publishing {MessageType} message for Order {OrderId}", 
                    messageType, GetOrderId(message));
                throw;
            }

            await Task.CompletedTask;
        }

        private static string GetOrderId(T message)
        {
            return message switch
            {
                OrderMessage order => order.OrderId,
                OrderStatusMessage status => status.OrderId,
                _ => "unknown"
            };
        }
    }
}
```

### 第4步：Order.API项目配置

appsettings.json

```
{
  "RabbitMQ": {
    "HostName": "localhost",
    "UserName": "myuser",
    "Password": "mypassword",
    "Port": 5672,
    "VirtualHost": "/"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

Program.cs

```
using Order.API.Controllers;
using Order.API.Services;
using Order.Core.Models;
using Order.Infrastructure.Services;
using RabbitMQ.Client;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Configure RabbitMQ
builder.Services.AddSingleton(sp =>
{
    var configuration = sp.GetRequiredService();
    return new ConnectionFactory
    {
        HostName = configuration["RabbitMQ:HostName"],
        UserName = configuration["RabbitMQ:UserName"],
        Password = configuration["RabbitMQ:Password"],
        Port = int.Parse(configuration["RabbitMQ:Port"] ?? "5672"),
        VirtualHost = configuration["RabbitMQ:VirtualHost"] ?? "/",
        DispatchConsumersAsync = true
    };
});

// Register RabbitMQ services
builder.Services.AddSingleton();
builder.Services.AddScoped();
builder.Services.AddScoped();

// Add Health Checks
builder.Services.AddHealthChecks()
    .AddRabbitMQ(provider => 
    {
        var factory = provider.GetRequiredService();
        return factory.CreateConnection();
    }, name: "rabbitmq");

// Add hosted service for status updates consumer
builder.Services.AddHostedService();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

// Add health check endpoint
app.MapHealthChecks("/health");

app.Run();
```

Services/IOrderService.cs

```
using Order.Core.Models;

namespace Order.API.Services
{
    public interface IOrderService
    {
        Task CreateOrderAsync(string customerId, string productId, int quantity, decimal unitPrice);
        Task GetOrderAsync(string orderId);
        Task UpdateOrderStatusAsync(string orderId, OrderStatus status);
    }
}
```

Services/OrderService.cs

![]()![]()

```
using Order.Core.Messages;
using Order.Core.Models;
using Order.Infrastructure.Services;

namespace Order.API.Services
{
    public class OrderService : IOrderService
    {
        private readonly IOrderPublisher _orderPublisher;
        private readonly ILogger _logger;
        
        // 内存存储用于演示（生产环境应该用数据库）
        private static readonly Dictionary<string, Order> _orders = new();

        public OrderService(IOrderPublisher orderPublisher, ILogger logger)
        {
            _orderPublisher = orderPublisher;
            _logger = logger;
        }

        public async Task CreateOrderAsync(string customerId, string productId, int quantity, decimal unitPrice)
        {
            var order = new Order
            {
                CustomerId = customerId,
                ProductId = productId,
                Quantity = quantity,
                TotalAmount = quantity * unitPrice,
                Status = OrderStatus.Pending
            };

            // 保存到内存
            _orders[order.Id] = order;

            // 发布订单创建事件
            var orderMessage = new OrderMessage
            {
                OrderId = order.Id,
                CustomerId = order.CustomerId,
                ProductId = order.ProductId,
                Quantity = order.Quantity,
                TotalAmount = order.TotalAmount,
                Action = "create"
            };

            await _orderPublisher.PublishOrderCreatedAsync(orderMessage);
            
            _logger.LogInformation("Order {OrderId} created and published", order.Id);

            return order;
        }

        public Task GetOrderAsync(string orderId)
        {
            _orders.TryGetValue(orderId, out var order);
            return Task.FromResult(order);
        }

        public async Task UpdateOrderStatusAsync(string orderId, OrderStatus status)
        {
            if (_orders.TryGetValue(orderId, out var order))
            {
                order.Status = status;
                order.ProcessedAt = DateTime.UtcNow;
                
                // 发布状态更新
                var statusMessage = new OrderStatusMessage
                {
                    OrderId = orderId,
                    Status = status,
                    Message = $"Order {status.ToString().ToLower()}"
                };

                await _orderPublisher.PublishOrderStatusAsync(statusMessage);
                
                _logger.LogInformation("Order {OrderId} status updated to {Status}", orderId, status);
            }
        }
    }
}
```

View Code

Services/OrderStatusConsumerService.cs

![]()![]()

```
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Options;
using Order.API.Services;
using Order.Core.Messages;
using Order.Infrastructure.Services;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

namespace Order.API.Services
{
    public class OrderStatusConsumerService : BackgroundService
    {
        private readonly IRabbitMQConnection _connection;
        private readonly IServiceProvider _serviceProvider;
        private readonly ILogger _logger;
        private IModel _channel;
        private const string QueueName = "order.status.queue";

        public OrderStatusConsumerService(
            IRabbitMQConnection connection,
            IServiceProvider serviceProvider,
            ILogger logger)
        {
            _connection = connection;
            _serviceProvider = serviceProvider;
            _logger = logger;
            InitializeChannel();
        }

        private void InitializeChannel()
        {
            if (!_connection.IsConnected)
            {
                _connection.TryConnect();
            }

            _channel = _connection.CreateModel();
            
            // 确保队列存在（已经在Publisher中声明，这里做双重保险）
            _channel.QueueDeclare(QueueName, durable: true, exclusive: false, autoDelete: false);
            
            _channel.BasicQos(0, 1, false); // 公平分发
            
            _logger.LogInformation("OrderStatusConsumerService channel initialized");
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            stoppingToken.ThrowIfCancellationRequested();

            var consumer = new AsyncEventingBasicConsumer(_channel);
            consumer.Received += async (model, ea) =>
            {
                var body = ea.Body.ToArray();
                var message = Encoding.UTF8.GetString(body);

                try
                {
                    await ProcessMessageAsync(message);
                    _channel.BasicAck(ea.DeliveryTag, false);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing message: {Message}", message);
                    _channel.BasicNack(ea.DeliveryTag, false, false); // 不重新入队
                }
            };

            _channel.BasicConsume(QueueName, false, consumer);

            _logger.LogInformation("OrderStatusConsumerService started consuming");

            await Task.CompletedTask;
        }

        private async Task ProcessMessageAsync(string message)
        {
            using var scope = _serviceProvider.CreateScope();
            var orderService = scope.ServiceProvider.GetRequiredService();

            try
            {
                var statusMessage = JsonSerializer.Deserialize(message);
                if (statusMessage != null)
                {
                    // 这里可以处理状态更新，比如更新数据库、发送通知等
                    _logger.LogInformation("Received order status update: {OrderId} -> {Status}", 
                        statusMessage.OrderId, statusMessage.Status);
                    
                    // 在实际应用中，这里可能会更新数据库中的订单状态
                    // await orderService.UpdateOrderStatusAsync(statusMessage.OrderId, statusMessage.Status);
                }
            }
            catch (JsonException ex)
            {
                _logger.LogError(ex, "Error deserializing message: {Message}", message);
                throw;
            }
        }

        public override void Dispose()
        {
            _channel?.Close();
            _channel?.Dispose();
            base.Dispose();
        }
    }
}
```

View Code

Controllers/OrdersController.cs

![]()![]()

```
using Microsoft.AspNetCore.Mvc;
using Order.API.Services;
using Order.Core.Models;

namespace Order.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class OrdersController : ControllerBase
    {
        private readonly IOrderService _orderService;
        private readonly ILogger _logger;

        public OrdersController(IOrderService orderService, ILogger logger)
        {
            _orderService = orderService;
            _logger = logger;
        }

        [HttpPost]
        public async Task> CreateOrder([FromBody] CreateOrderRequest request)
        {
            try
            {
                var order = await _orderService.CreateOrderAsync(
                    request.CustomerId, 
                    request.ProductId, 
                    request.Quantity, 
                    request.UnitPrice);

                return Ok(order);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error creating order");
                return StatusCode(500, "Error creating order");
            }
        }

        [HttpGet("{orderId}")]
        public async Task> GetOrder(string orderId)
        {
            var order = await _orderService.GetOrderAsync(orderId);
            if (order == null)
            {
                return NotFound();
            }
            return Ok(order);
        }

        [HttpGet]
        public ActionResult> GetOrders()
        {
            // 这里只是演示，实际应该从数据库获取
            return Ok(new List());
        }
    }

    public class CreateOrderRequest
    {
        public string CustomerId { get; set; }
        public string ProductId { get; set; }
        public int Quantity { get; set; }
        public decimal UnitPrice { get; set; }
    }
}
```

View Code

### 第5步：订单处理器服务（OrderProcessor.Service）

Program.cs

![]()![]()

```
using Order.Core.Messages;
using Order.Infrastructure.Services;
using OrderProcessor.Service.Services;
using RabbitMQ.Client;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddHostedService();

// Configure RabbitMQ
builder.Services.AddSingleton(sp =>
{
    var configuration = sp.GetRequiredService();
    return new ConnectionFactory
    {
        HostName = configuration["RabbitMQ:HostName"],
        UserName = configuration["RabbitMQ:UserName"],
        Password = configuration["RabbitMQ:Password"],
        Port = int.Parse(configuration["RabbitMQ:Port"] ?? "5672"),
        VirtualHost = configuration["RabbitMQ:VirtualHost"] ?? "/",
        DispatchConsumersAsync = true
    };
});

builder.Services.AddSingleton();
builder.Services.AddScoped();

builder.Services.AddLogging();

var host = builder.Build();
host.Run();
```

View Code

Services/OrderProcessorService.cs

![]()![]()

```
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Logging;
using Order.Core.Messages;
using Order.Infrastructure.Services;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

namespace OrderProcessor.Service.Services
{
    public class OrderProcessorService : BackgroundService
    {
        private readonly IRabbitMQConnection _connection;
        private readonly IOrderPublisher _orderPublisher;
        private readonly ILogger _logger;
        private IModel _channel;
        private const string QueueName = "order.created.queue";

        public OrderProcessorService(
            IRabbitMQConnection connection,
            IOrderPublisher orderPublisher,
            ILogger logger)
        {
            _connection = connection;
            _orderPublisher = orderPublisher;
            _logger = logger;
            InitializeChannel();
        }

        private void InitializeChannel()
        {
            if (!_connection.IsConnected)
            {
                _connection.TryConnect();
            }

            _channel = _connection.CreateModel();
            _channel.QueueDeclare(QueueName, durable: true, exclusive: false, autoDelete: false);
            _channel.BasicQos(0, 1, false);
            
            _logger.LogInformation("OrderProcessorService channel initialized");
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            stoppingToken.ThrowIfCancellationRequested();

            var consumer = new AsyncEventingBasicConsumer(_channel);
            consumer.Received += async (model, ea) =>
            {
                var body = ea.Body.ToArray();
                var message = Encoding.UTF8.GetString(body);

                try
                {
                    await ProcessOrderAsync(message);
                    _channel.BasicAck(ea.DeliveryTag, false);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing order: {Message}", message);
                    _channel.BasicNack(ea.DeliveryTag, false, false);
                }
            };

            _channel.BasicConsume(QueueName, false, consumer);

            _logger.LogInformation("OrderProcessorService started consuming orders");

            await Task.CompletedTask;
        }

        private async Task ProcessOrderAsync(string message)
        {
            try
            {
                var orderMessage = JsonSerializer.Deserialize(message);
                if (orderMessage == null)
                {
                    _logger.LogWarning("Received invalid order message: {Message}", message);
                    return;
                }

                _logger.LogInformation("Processing order {OrderId} for customer {CustomerId}", 
                    orderMessage.OrderId, orderMessage.CustomerId);

                // 模拟订单处理逻辑
                await ProcessOrderBusinessLogic(orderMessage);

                // 发布处理完成状态
                var statusMessage = new OrderStatusMessage
                {
                    OrderId = orderMessage.OrderId,
                    Status = Order.Core.Models.OrderStatus.Completed,
                    Message = "Order processed successfully"
                };

                await _orderPublisher.PublishOrderStatusAsync(statusMessage);
                
                _logger.LogInformation("Order {OrderId} processed successfully", orderMessage.OrderId);
            }
            catch (JsonException ex)
            {
                _logger.LogError(ex, "Error deserializing order message: {Message}", message);
                throw;
            }
        }

        private async Task ProcessOrderBusinessLogic(OrderMessage orderMessage)
        {
            // 模拟复杂的业务逻辑处理
            _logger.LogInformation("Starting business logic for order {OrderId}", orderMessage.OrderId);
            
            // 模拟处理时间
            var random = new Random();
            var processingTime = random.Next(2000, 5000);
            await Task.Delay(processingTime);
            
            // 模拟10%的失败率
            if (random.Next(0, 10) == 0)
            {
                throw new Exception("Simulated business logic failure");
            }
            
            _logger.LogInformation("Business logic completed for order {OrderId}", orderMessage.OrderId);
        }

        public override void Dispose()
        {
            _channel?.Close();
            _channel?.Dispose();
            base.Dispose();
        }
    }
}
```

View Code

### 第6步：运行与测试

1. 启动服务

   ```
   # 终端1：启动Order.API
   cd Order.API
   dotnet run

   # 终端2：启动OrderProcessor.Service
   cd OrderProcessor.Service
   dotnet run
   ```
2. 测试API

   ```
   # 创建订单
   curl -X POST "https://localhost:7000/api/orders" \
        -H "Content-Type: application/json" \
        -d '{
          "customerId": "customer-123",
          "productId": "product-456", 
          "quantity": 2,
          "unitPrice": 29.99
        }'

   # 查询订单状态
   curl "https://localhost:7000/api/orders/{orderId}"
   ```
3. 测试健康检查

   ```
   GET https://localhost:7000/health
   ```
4. 观察日志输出

   * Order.API：接收HTTP请求，发布订单创建消息
   * OrderProcessor.Service：消费订单消息，处理业务逻辑，发布状态更新
   * Order.API：消费状态更新消息
5. 测试错误场景

   * 停止RabbitMQ服务，观察重连机制
   * 停止OrderProcessor.Service，观察消息堆积
   * 重启服务，观察消息恢复处理

### 第7步：高级特性 - 配置重试和 resilience

在Order.Infrastructure中添加Polly支持：

```
// 添加NuGet包
dotnet add package Polly
dotnet add package Microsoft.Extensions.Http.Polly

// 在Program.cs中添加重试策略
builder.Services.AddHttpClient("retry-client")
    .AddTransientHttpErrorPolicy(policy => 
        policy.WaitAndRetryAsync(3, retryAttempt => 
            TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));
```

---

## 本章总结

在这一章中，我们成功地将RabbitMQ集成到ASP.NET Core应用程序中，构建了一个完整的微服务系统：

1. 依赖注入配置：正确管理RabbitMQ连接和通道的生命周期。
2. 托管服务：使用`BackgroundService`实现长时间运行的消费者服务。
3. 领域驱动设计：采用分层架构，分离关注点。
4. 消息序列化：使用JSON序列化消息体。
5. 健康检查：集成RabbitMQ健康监控。
6. 错误处理：实现完善的错误处理和日志记录。
7. 配置管理：从配置文件读取连接字符串。

这个架构为构建生产级的微服务系统提供了坚实的基础。在下一章，我们将学习RabbitMQ的监控、治理和生产环境的最佳实践。
