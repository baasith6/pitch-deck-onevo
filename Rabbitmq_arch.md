# RabbitMQ Messaging Architecture - Complete Guide

## Table of Contents

1. [What is RabbitMQ?](#what-is-rabbitmq)
2. [Why We Use RabbitMQ](#why-we-use-rabbitmq)
3. [Architecture Overview](#architecture-overview)
4. [Implementation Details](#implementation-details)
5. [Event Types & Queues](#event-types--queues)
6. [Publisher Implementation](#publisher-implementation)
7. [Consumer Implementation](#consumer-implementation)
8. [Background Services](#background-services)
9. [Event Flow Examples](#event-flow-examples)
10. [Configuration](#configuration)
11. [Troubleshooting](#troubleshooting)

---

## What is RabbitMQ?

RabbitMQ is an **open-source message broker** that implements the Advanced Message Queuing Protocol (AMQP). It acts as a middleman between services, allowing them to communicate asynchronously.

### Key Concepts

1. **Message**: Data sent from one service to another
2. **Queue**: Buffer that stores messages until they're consumed
3. **Exchange**: Routes messages to queues (we use default exchange)
4. **Publisher**: Service that sends messages
5. **Consumer**: Service that receives and processes messages

### Simple Analogy

Think of RabbitMQ like a **post office**:
- **Publisher** = Person sending mail
- **Queue** = Post office box
- **Consumer** = Person receiving mail
- **Message** = Letter/package

---

## Why We Use RabbitMQ

### Benefits

1. **Decoupling**: Services don't need to know about each other
   - ContentService doesn't need to know about NotificationService
   - They communicate through events

2. **Asynchronous Processing**: Non-blocking operations
   - Post publishing doesn't wait for notifications
   - Analytics updates happen in background

3. **Reliability**: Messages are persisted
   - If service is down, messages wait in queue
   - No data loss

4. **Scalability**: Easy to add new consumers
   - Add new service that listens to events
   - No changes to existing services

5. **Flexibility**: Easy to add new features
   - New service can consume existing events
   - No breaking changes

### Without RabbitMQ (Synchronous)

```
User Publishes Post
    │
    ▼
ContentService
    │
    ├──► Calls NotificationService (wait)
    ├──► Calls AnalyticsService (wait)
    └──► Returns response (slow!)
```

### With RabbitMQ (Asynchronous)

```
User Publishes Post
    │
    ▼
ContentService
    │
    ├──► Publishes Event (instant)
    └──► Returns response (fast!)
         │
         ▼
    RabbitMQ Queue
         │
         ├──► NotificationService (processes in background)
         └──► AnalyticsService (processes in background)
```

---

## Architecture Overview

### Message Flow

```
┌─────────────────┐
│  Service A      │
│  (Publisher)     │
└────────┬────────┘
         │
         │ Publish Event
         ▼
┌─────────────────┐
│   RabbitMQ      │
│   Queue         │
│   "post.published"│
└────────┬────────┘
         │
         │ Consume Event
         ├──────────────┐
         │              │
         ▼              ▼
┌─────────────┐  ┌─────────────┐
│ Service B   │  │ Service C   │
│ (Consumer)  │  │ (Consumer)  │
└─────────────┘  └─────────────┘
```

### Our Implementation

```
┌─────────────────────────────────────┐
│         API Gateway                 │
│  (Single Process, All Services)     │
└─────────────────────────────────────┘
         │
         │ Shared RabbitMQ Connection
         ▼
┌─────────────────────────────────────┐
│            RabbitMQ                 │
│  ┌──────────────────────────────┐  │
│  │ Queue: post.published         │  │
│  │ Queue: post.approved          │  │
│  │ Queue: socialaccount.connected│  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
         │
         │ Background Services Consume
         ├─────────────────┬───────────┐
         ▼                 ▼           ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│Notification │  │  Analytics   │  │  Scheduler   │
│Consumer     │  │  Consumer    │  │  Consumer    │
└──────────────┘  └──────────────┘  └──────────────┘
```

---

## Implementation Details

### Shared Messaging Library

**Location**: `NexusPost/src/Shared/Shared.Messaging/`

**Files**:
1. `IMessagePublisher.cs` - Interface for publishing
2. `IMessageConsumer.cs` - Interface for consuming
3. `RabbitMQPublisher.cs` - Implementation
4. `RabbitMQConsumer.cs` - Implementation
5. `ServiceCollectionExtensions.cs` - Configuration

### Publisher Interface

```csharp
public interface IMessagePublisher
{
    Task PublishAsync<T>(string queueName, T message);
    Task PublishAsync<T>(string exchange, string routingKey, T message);
}
```

### Consumer Interface

```csharp
public interface IMessageConsumer
{
    Task StartConsumingAsync<T>(string queueName, Func<T, Task> handler);
    Task StopConsumingAsync();
}
```

---

## Event Types & Queues

### Complete Event List

| Event | Published By | Consumed By | Queue Name | Purpose |
|-------|-------------|-------------|------------|---------|
| `PostPublishedEvent` | ContentService | NotificationService, AnalyticsService | `post.published` | Post was published |
| `PostApprovedEvent` | ContentService | NotificationService, SchedulerService | `post.approved` | Post was approved |
| `PostCreatedEvent` | ContentService | (Future) | `post.created` | Post was created |
| `PostScheduledEvent` | ContentService | (Future) | `post.scheduled` | Post was scheduled |
| `SocialAccountConnectedEvent` | SocialAccountService | NotificationService | `socialaccount.connected` | Account connected |
| `SocialAccountDisconnectedEvent` | SocialAccountService | (Future) | `socialaccount.disconnected` | Account disconnected |
| `TokenExpiredEvent` | SocialAccountService | (Future) | `token.expired` | Token expired |
| `UserRegisteredEvent` | AuthService | (Future) | `user.registered` | User registered |
| `UserLoginEvent` | AuthService | (Future) | `user.login` | User logged in |
| `UserLogoutEvent` | AuthService | (Future) | `user.logout` | User logged out |
| `WebhookReceivedEvent` | WebhookService | (Future) | `webhook.received` | Webhook received |

### Event Definitions

#### PostPublishedEvent
```csharp
public class PostPublishedEvent
{
    public Guid PostId { get; set; }
    public Guid TenantId { get; set; }
    public Guid UserId { get; set; }
    public DateTime PublishedAt { get; set; }
    public List<string> Platforms { get; set; }
}
```

#### PostApprovedEvent
```csharp
public class PostApprovedEvent
{
    public Guid PostId { get; set; }
    public Guid ApprovalRequestId { get; set; }
    public Guid ReviewerId { get; set; }
    public Guid TenantId { get; set; }
}
```

#### SocialAccountConnectedEvent
```csharp
public class SocialAccountConnectedEvent
{
    public Guid SocialAccountId { get; set; }
    public Guid UserId { get; set; }
    public Guid TenantId { get; set; }
    public string Platform { get; set; }
    public DateTime ConnectedAt { get; set; }
}
```

---

## Publisher Implementation

### How to Publish an Event

**Location**: `NexusPost/src/Shared/Shared.Messaging/RabbitMQPublisher.cs`

**Example from ContentService**:
```csharp
// Inject IMessagePublisher
private readonly IMessagePublisher? _messagePublisher;

// Publish event
if (_messagePublisher != null)
{
    await _messagePublisher.PublishAsync("post.published", new PostPublishedEvent
    {
        PostId = post.Id,
        TenantId = post.TenantId,
        UserId = post.CreatedByUserId,
        PublishedAt = DateTime.UtcNow,
        Platforms = platforms
    });
}
```

### Publisher Implementation Details

```csharp
public class RabbitMQMessagePublisher : IMessagePublisher
{
    private readonly IConnection _connection;
    private readonly IModel _channel;
    
    public async Task PublishAsync<T>(string queueName, T message)
    {
        // 1. Serialize message to JSON
        var messageBody = Encoding.UTF8.GetBytes(
            JsonConvert.SerializeObject(message)
        );
        
        // 2. Create message properties
        var properties = _channel.CreateBasicProperties();
        properties.Persistent = true; // Message survives broker restart
        properties.MessageId = Guid.NewGuid().ToString();
        properties.Timestamp = new AmqpTimestamp(
            DateTimeOffset.UtcNow.ToUnixTimeSeconds()
        );
        
        // 3. Declare queue (creates if doesn't exist)
        _channel.QueueDeclare(
            queue: queueName,
            durable: true,      // Queue survives broker restart
            exclusive: false,
            autoDelete: false
        );
        
        // 4. Publish message
        _channel.BasicPublish(
            exchange: "",        // Default exchange
            routingKey: queueName,
            properties: properties,
            body: messageBody
        );
    }
}
```

### Key Points

1. **Persistent Messages**: `properties.Persistent = true`
   - Messages survive RabbitMQ server restart
   - Important for reliability

2. **Durable Queues**: `durable: true`
   - Queues survive RabbitMQ server restart
   - Messages are not lost

3. **JSON Serialization**: Uses Newtonsoft.Json
   - Messages are human-readable
   - Easy to debug

4. **Error Handling**: Wrapped in try-catch
   - Logs errors
   - Doesn't crash application if RabbitMQ is down

---

## Consumer Implementation

### How to Consume Events

**Location**: `NexusPost/src/Shared/Shared.Messaging/RabbitMQConsumer.cs`

**Example from NotificationService**:
```csharp
public class RabbitMQConsumerService : BackgroundService
{
    private readonly IMessageConsumer? _messageConsumer;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Start consuming events
        await _messageConsumer.StartConsumingAsync<PostPublishedEvent>(
            "post.published",
            async (eventData) =>
            {
                // Process event
                using var scope = _serviceProvider.CreateScope();
                var handler = scope.ServiceProvider
                    .GetRequiredService<PostPublishedEventHandler>();
                await handler.HandleAsync(eventData);
            }
        );
    }
}
```

### Consumer Implementation Details

```csharp
public class RabbitMQMessageConsumer : IMessageConsumer
{
    public async Task StartConsumingAsync<T>(
        string queueName, 
        Func<T, Task> handler)
    {
        // 1. Declare queue
        _channel.QueueDeclare(
            queue: queueName,
            durable: true,
            exclusive: false,
            autoDelete: false
        );
        
        // 2. Create consumer
        var consumer = new EventingBasicConsumer(_channel);
        
        // 3. Handle received messages
        consumer.Received += (model, ea) =>
        {
            try
            {
                // Deserialize message
                var body = ea.Body.ToArray();
                var message = JsonConvert.DeserializeObject<T>(
                    Encoding.UTF8.GetString(body)
                );
                
                if (message != null)
                {
                    // Process message
                    Task.Run(async () =>
                    {
                        try
                        {
                            await handler(message);
                            
                            // Acknowledge message (remove from queue)
                            _channel.BasicAck(
                                deliveryTag: ea.DeliveryTag,
                                multiple: false
                            );
                        }
                        catch (Exception ex)
                        {
                            // Reject and requeue message
                            _channel.BasicNack(
                                deliveryTag: ea.DeliveryTag,
                                multiple: false,
                                requeue: true  // Put back in queue
                            );
                        }
                    });
                }
            }
            catch (Exception ex)
            {
                // Deserialization error - reject message
                _channel.BasicNack(
                    deliveryTag: ea.DeliveryTag,
                    multiple: false,
                    requeue: true
                );
            }
        };
        
        // 4. Start consuming
        _channel.BasicConsume(
            queue: queueName,
            autoAck: false,  // Manual acknowledgment
            consumer: consumer
        );
    }
}
```

### Key Points

1. **Manual Acknowledgment**: `autoAck: false`
   - Message is removed from queue only after processing
   - If processing fails, message is requeued

2. **Error Handling**: Try-catch around handler
   - Errors don't crash consumer
   - Failed messages are requeued for retry

3. **Scoped Services**: Creates scope for each message
   - Each message gets fresh service instances
   - Prevents state leakage between messages

4. **Async Processing**: Uses `Task.Run`
   - Handles async handlers properly
   - Doesn't block consumer thread

---

## Background Services

### Services That Consume Events

#### 1. NotificationService.RabbitMQConsumerService

**Location**: `NexusPost/src/Services/NotificationService/Notification.API/Services/RabbitMQConsumerService.cs`

**Consumes**:
- `post.published` → Creates notification
- `post.approved` → Creates notification
- `socialaccount.connected` → Creates welcome notification

**Implementation**:
```csharp
public class RabbitMQConsumerService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Subscribe to post.published
        await _messageConsumer.StartConsumingAsync<PostPublishedEvent>(
            "post.published",
            async (eventData) =>
            {
                using var scope = _serviceProvider.CreateScope();
                var handler = scope.ServiceProvider
                    .GetRequiredService<PostPublishedEventHandler>();
                await handler.HandleAsync(eventData);
            }
        );
        
        // Subscribe to post.approved
        await _messageConsumer.StartConsumingAsync<PostApprovedEvent>(
            "post.approved",
            async (eventData) =>
            {
                using var scope = _serviceProvider.CreateScope();
                var handler = scope.ServiceProvider
                    .GetRequiredService<PostApprovedEventHandler>();
                await handler.HandleAsync(eventData);
            }
        );
        
        // Keep service running
        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

#### 2. AnalyticsService.RabbitMQConsumerService

**Location**: `NexusPost/src/Services/AnalyticsService/Analytics/Analytics.API/Services/RabbitMQConsumerService.cs`

**Consumes**:
- `post.published` → Creates analytics record

**Implementation**:
```csharp
await _messageConsumer.StartConsumingAsync<PostPublishedEvent>(
    "post.published",
    async (eventData) =>
    {
        // Create analytics record for each platform
        var targets = await GetPostTargetsAsync(eventData.PostId);
        
        foreach (var target in targets)
        {
            var analyticsRecord = new AnalyticsRecord {
                PostId = eventData.PostId,
                Platform = target.Platform,
                Status = "Active"
            };
            
            await _analyticsRecordRepository.AddAsync(analyticsRecord);
        }
    }
);
```

#### 3. SchedulerService.RabbitMQConsumerService

**Location**: `NexusPost/src/Services/SchedulerService/Scheduler.Application/Services/RabbitMQConsumerService.cs`

**Consumes**:
- `post.approved` → If post has ScheduledAt, creates scheduled job
- `post.created.from.webhook` → Schedules webhook-created posts

**Implementation**:
```csharp
await _messageConsumer.StartConsumingAsync<PostApprovedEvent>(
    "post.approved.queue",
    async (eventData) =>
    {
        var post = await GetPostAsync(eventData.PostId);
        
        // If post is scheduled, create scheduled job
        if (post.ScheduledAt.HasValue && post.ScheduledAt > DateTime.UtcNow)
        {
            await _schedulerService.SchedulePostAsync(new SchedulePostRequest {
                PostId = post.Id,
                ScheduledAt = post.ScheduledAt.Value,
                TenantId = post.TenantId
            });
        }
    }
);
```

---

## Event Flow Examples

### Example 1: Post Publishing Flow

```
1. User Clicks "Publish Now"
   │
   ▼
2. POST /api/posts/{id}/publish
   ContentService.PublishPostAsync()
   │
   ├──► Publishes to platforms (Facebook, Instagram, etc.)
   │
   └──► Publishes PostPublishedEvent to RabbitMQ
        Queue: "post.published"
        │
        ├──► NotificationService (Consumes)
        │    │
        │    └──► Creates Notification
        │         ├──► Stores in Notifications table
        │         └──► Sends via SignalR (real-time)
        │
        └──► AnalyticsService (Consumes)
             │
             └──► Creates AnalyticsRecord
                  ├──► Stores in AnalyticsServiceRecords table
                  └──► Initializes metrics to zero
```

**Timeline**:
- T+0ms: User clicks publish
- T+50ms: Post published to platforms
- T+100ms: PostPublishedEvent published to RabbitMQ
- T+150ms: NotificationService consumes event
- T+200ms: AnalyticsService consumes event
- T+250ms: User sees notification (SignalR)

### Example 2: Post Approval Flow

```
1. Reviewer Approves Post
   │
   ▼
2. POST /api/approvals/review
   ContentService.ReviewApprovalAsync()
   │
   ├──► Updates ApprovalRequest status = "Approved"
   ├──► Updates Post status = "Approved"
   │
   └──► Publishes PostApprovedEvent to RabbitMQ
        Queue: "post.approved"
        │
        ├──► NotificationService (Consumes)
        │    │
        │    └──► Creates Notification
        │         "Your post has been approved!"
        │
        └──► SchedulerService (Consumes)
             │
             └──► Checks if post has ScheduledAt
                  │
                  ├──► If yes: Creates ScheduledJob
                  │    └──► JobProcessorService will process later
                  │
                  └──► If no: Does nothing
```

### Example 3: Social Account Connection Flow

```
1. User Connects Facebook Account
   │
   ▼
2. OAuth Callback Success
   SocialAccountService.HandleOAuthCallbackAsync()
   │
   ├──► Creates SocialAccount record
   ├──► Encrypts and stores tokens
   │
   └──► Publishes SocialAccountConnectedEvent
        Queue: "socialaccount.connected"
        │
        └──► NotificationService (Consumes)
             │
             └──► Creates Notification
                  "Facebook account connected successfully!"
                  └──► Sends welcome email (if configured)
```

---

## Configuration

### Connection Settings

**File**: `appsettings.json` or `appsettings.Development.json`

```json
{
  "RabbitMq": {
    "Host": "localhost",
    "Username": "guest",
    "Password": "guest",
    "Port": 5672
  }
}
```

### Disabling RabbitMQ

To disable RabbitMQ (for development without RabbitMQ):

```json
{
  "RabbitMq": {
    "Host": "disabled"
  }
}
```

When disabled:
- Publishers return null (no errors)
- Consumers don't start
- Application runs without messaging

### Docker Setup

**File**: `NexusPost/docker/docker-compose.yml`

```yaml
rabbitmq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"   # Main service
    - "15672:15672" # Management UI
  environment:
    - RABBITMQ_DEFAULT_USER=guest
    - RABBITMQ_DEFAULT_PASS=guest
```

**Start RabbitMQ**:
```bash
cd NexusPost/docker
docker-compose up -d rabbitmq
```

**Access Management UI**:
- URL: `http://localhost:15672`
- Username: `guest`
- Password: `guest`

### Connection Factory Settings

**Location**: `NexusPost/src/Shared/Shared.Messaging/ServiceCollectionExtensions.cs`

```csharp
var factory = new ConnectionFactory
{
    HostName = host,
    UserName = username,
    Password = password,
    Port = 5672,
    VirtualHost = "/",
    
    // Timeout settings
    RequestedConnectionTimeout = TimeSpan.FromSeconds(10),
    SocketReadTimeout = TimeSpan.FromSeconds(10),
    SocketWriteTimeout = TimeSpan.FromSeconds(10),
    
    // Automatic recovery
    AutomaticRecoveryEnabled = true,
    NetworkRecoveryInterval = TimeSpan.FromSeconds(10)
};
```

**Key Settings**:
- **AutomaticRecoveryEnabled**: Reconnects if connection lost
- **NetworkRecoveryInterval**: How often to retry connection
- **Timeouts**: Prevents hanging connections

---

## Troubleshooting

### Common Issues

#### 1. RabbitMQ Connection Failed

**Error**: `Connection refused` or `SocketException`

**Solutions**:
1. Check if RabbitMQ is running: `docker ps`
2. Check port 5672 is not blocked
3. Verify connection string in appsettings.json
4. Check firewall rules

#### 2. Messages Not Being Consumed

**Symptoms**: Messages in queue but not processed

**Solutions**:
1. Check if background service is running
2. Check logs for consumer errors
3. Verify queue name matches
4. Check if consumer is subscribed to correct queue

#### 3. Messages Being Lost

**Symptoms**: Events published but not processed

**Solutions**:
1. Ensure queues are durable: `durable: true`
2. Ensure messages are persistent: `Persistent = true`
3. Check if consumer acknowledges messages
4. Verify RabbitMQ is not restarting

#### 4. High Memory Usage

**Symptoms**: RabbitMQ using too much memory

**Solutions**:
1. Check queue lengths (messages waiting)
2. Ensure consumers are processing messages
3. Increase consumer instances
4. Set message TTL (time-to-live)

### Monitoring

#### RabbitMQ Management UI

**Access**: `http://localhost:15672`

**Features**:
- View queues and message counts
- See message rates (publish/consume)
- Monitor connections
- View queue contents
- Check consumer status

#### Logging

**Location**: Application logs

**What to Look For**:
- `Message published to {Queue}` - Publisher working
- `Started consuming messages from queue {Queue}` - Consumer started
- `Message processed successfully` - Consumer working
- `Error publishing message` - Publisher error
- `Error processing message` - Consumer error

---

## Best Practices

### 1. Idempotency

**Problem**: Message might be processed twice (if requeued)

**Solution**: Make handlers idempotent
```csharp
// Check if already processed
var existingRecord = await _repository.GetByPostIdAsync(eventData.PostId);
if (existingRecord != null) {
    return; // Already processed
}

// Process
await CreateRecordAsync(eventData);
```

### 2. Error Handling

**Always wrap handlers in try-catch**:
```csharp
consumer.Received += async (model, ea) =>
{
    try
    {
        await handler(message);
        _channel.BasicAck(ea.DeliveryTag, false);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error processing message");
        _channel.BasicNack(ea.DeliveryTag, false, requeue: true);
    }
};
```

### 3. Message Size

**Keep messages small**:
- Store IDs, not full objects
- Fetch data in consumer if needed
- Use references, not embedded data

### 4. Queue Naming

**Use clear, descriptive names**:
- ✅ `post.published`
- ✅ `post.approved`
- ❌ `events`
- ❌ `queue1`

### 5. Monitoring

**Monitor queue lengths**:
- High queue length = consumers too slow
- Zero queue length = everything working
- Growing queue = need more consumers

---

## Summary

### Key Takeaways

1. **RabbitMQ enables asynchronous communication** between services
2. **Events are published** when actions occur (post published, approved, etc.)
3. **Background services consume events** and process them
4. **Messages are persistent** and won't be lost
5. **Services are decoupled** - they don't need to know about each other

### Event Flow Summary

```
Action → Service → Publish Event → RabbitMQ Queue → Consumer → Process
```

### Current Implementation

- **3 Background Services** consuming events
- **10+ Event Types** defined
- **Automatic Recovery** if RabbitMQ restarts
- **Error Handling** with retry logic
- **Persistent Messages** for reliability

---

**Last Updated**: December 2024
**Version**: 1.0.0

