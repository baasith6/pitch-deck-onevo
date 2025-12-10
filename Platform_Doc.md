# Onevo - Complete Platform Documentation

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [System Architecture](#system-architecture)
3. [Entry Point & How to Run](#entry-point--how-to-run)
4. [Services Overview](#services-overview)
5. [Database Schema](#database-schema)
6. [RabbitMQ Messaging](#rabbitmq-messaging)
7. [API Endpoints](#api-endpoints)
8. [Feature Implementation Details](#feature-implementation-details)

---

## Architecture Overview

### System Type
**Modular Monolith Architecture** - All services run in a single process through the API Gateway

### Technology Stack
- **Backend**: .NET 8, C#
- **Frontend**: Angular 17, TypeScript
- **Database**: SQL Server (Single shared database)
- **Messaging**: RabbitMQ (Event-driven communication)
- **Cache**: Redis (Rate limiting, caching)
- **Payment**: Stripe
- **AI**: OpenAI GPT-4, Anthropic Claude
- **Infrastructure**: Azure Cloud, Docker-ready

### Key Architectural Decisions

1. **Single Database (Modular Monolith)**
   - All services share `NexusPostDbContext`
   - Simplifies migrations and data consistency
   - Single source of truth for all data

2. **API Gateway Pattern**
   - Single entry point: `ApiGateway` project
   - All controllers registered in one process
   - Port: 5000 (HTTP), 5001 (HTTPS)

3. **Event-Driven Architecture**
   - RabbitMQ for async communication
   - Services publish events, other services consume
   - Decoupled service communication

4. **Shared Infrastructure**
   - `Shared.Kernel`: Base entities, interfaces
   - `Shared.Contracts`: Common DTOs, events
   - `Shared.Infrastructure`: Database context, repositories
   - `Shared.Messaging`: RabbitMQ publisher/consumer

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway (Port 5000)                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  All Controllers (31 Controllers, 195 Endpoints)     │  │
│  │  - AuthController, PostsController, etc.              │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Background Services                                  │  │
│  │  - RabbitMQ Consumers                                 │  │
│  │  - Job Processors                                     │  │
│  │  - Email Queue Processor                              │  │
│  │  - Analytics Refresh Service                          │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  SQL Server  │   │   RabbitMQ   │   │    Redis     │
│  (Database)  │   │  (Messaging) │   │   (Cache)    │
└──────────────┘   └──────────────┘   └──────────────┘
```

### Service Communication Flow

```
User Request
    │
    ▼
API Gateway (Port 5000)
    │
    ├──► Direct Service Call (Synchronous)
    │    └──► Returns Response
    │
    └──► Event Publishing (Asynchronous)
         └──► RabbitMQ Queue
              └──► Background Service Consumes
                  └──► Processes Event
                      └──► Updates Database
```

---

## Entry Point & How to Run

### Single Entry Point
**File**: `NexusPost/src/ApiGateway/Program.cs`

This is the **ONLY** entry point for running the entire application. All services are hosted in this single process.

### How to Run Locally

#### Prerequisites
1. .NET 8 SDK
2. SQL Server (Local or Docker)
3. RabbitMQ (Docker recommended)
4. Redis (Optional, for rate limiting)

#### Step 1: Start Infrastructure (Docker)
```bash
cd NexusPost/docker
docker-compose up -d
```

This starts:
- SQL Server (Port 1433)
- RabbitMQ (Port 5672, Management UI: 15672)
- Redis (Port 6379)

#### Step 2: Configure Connection Strings
Edit `NexusPost/src/ApiGateway/appsettings.Development.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=NexusPost;User Id=sa;Password=Your_strong_password123!;TrustServerCertificate=True;Encrypt=False;"
  },
  "RabbitMq": {
    "Host": "localhost",
    "Username": "guest",
    "Password": "guest"
  }
}
```

#### Step 3: Run the Application
```bash
cd NexusPost/src/ApiGateway
dotnet run
```

The application will:
1. ✅ Connect to SQL Server
2. ✅ Run database migrations automatically
3. ✅ Seed admin user
4. ✅ Seed billing plans
5. ✅ Connect to RabbitMQ
6. ✅ Start all background services
7. ✅ Start API Gateway on `http://localhost:5000`

### Access Points

- **API**: `http://localhost:5000`
- **Swagger**: `http://localhost:5000/swagger`
- **Health Check**: `http://localhost:5000/health`
- **RabbitMQ Management**: `http://localhost:15672` (guest/guest)

### What Happens on Startup

1. **Database Migration**
   - Checks for pending migrations
   - Runs migrations automatically
   - Creates all tables if they don't exist

2. **Service Registration**
   - Registers all 11 services
   - Registers all repositories
   - Registers all application services

3. **Background Services Start**
   - RabbitMQ consumers for each service
   - Job processor for scheduled posts
   - Email queue processor
   - Analytics refresh service

4. **Seed Data**
   - Creates admin user (if not exists)
   - Creates billing plans (Basic, Pro, Enterprise, Agency)

---

## Services Overview

### Total: 11 Services

| # | Service | Purpose | Controllers | Endpoints |
|---|---------|---------|-------------|-----------|
| 1 | **AuthService** | Authentication, authorization, user & tenant management | 3 | 32 |
| 2 | **ContentService** | Post creation, media management, approval workflows | 4 | 27 |
| 3 | **SocialAccountService** | OAuth connections, social media account management | 2 | 16 |
| 4 | **SchedulerService** | Post scheduling, background job processing | 2 | 12 |
| 5 | **AnalyticsService** | Post performance tracking, engagement metrics | 1 | 7 |
| 6 | **AIService** | Content generation, sentiment analysis, hashtag suggestions | 1 | 9 |
| 7 | **NotificationService** | Multi-channel notifications (In-app, Email, Push) | 3 | 15 |
| 8 | **CollabService** | Team collaboration, workspaces, comments, tasks | 5 | 21 |
| 9 | **BillingService** | Stripe integration, subscriptions, invoices | 5 | 9 |
| 10 | **WebhookService** | Incoming webhook handling, subscriptions | 2 | 8 |
| 11 | **DeveloperAPIService** | API key management, developer apps, rate limiting | 3 | 8 |

**Total**: 31 Controllers, 195 API Endpoints

---

## Database Schema

### Database Name
`NexusPost` (Single shared database)

### Total Tables: 60+

#### Auth Service Tables (4)
1. **Tenants** - Multi-tenant organization data
2. **Users** - User accounts (individuals, team members, client users)
3. **RefreshTokens** - JWT refresh token management
4. **UserSettings** - User preferences and settings

#### Social Account Service Tables (4)
5. **SocialAccounts** - Connected social media accounts
6. **SocialAccountPages** - Facebook pages, Instagram accounts, etc.
7. **SocialAccountTokens** - Encrypted OAuth tokens
8. **OAuthStates** - OAuth flow state management

#### Content Service Tables (9)
9. **Clients** - Client accounts (for agencies)
10. **Posts** - Social media posts
11. **MediaAssets** - Uploaded media files
12. **PostVersions** - Post version history
13. **PostTargets** - Target platforms for posts
14. **ApprovalRequests** - Post approval workflow
15. **ContentScheduledJobs** - Scheduled post jobs
16. **ContentPublishLogs** - Publishing history
17. **ContentAnalyticsRecords** - Basic analytics

#### Billing Service Tables (5)
18. **BillingPlans** - Subscription plans (Basic, Pro, Enterprise, Agency)
19. **Subscriptions** - Active subscriptions
20. **PaymentMethods** - Stripe payment methods
21. **Invoices** - Billing invoices
22. **TrialUsages** - Free trial tracking

#### Collab Service Tables (6)
23. **Workspaces** - Team workspaces
24. **TeamMembers** - Workspace team members
25. **Comments** - Post/workspace comments
26. **Tasks** - Task assignments
27. **CollaborationRequests** - Collaboration invites
28. **ActivityLogs** - Activity tracking

#### AI Service Tables (4)
29. **AIGenerations** - AI-generated content
30. **SentimentAnalyses** - Sentiment analysis results
31. **HashtagSuggestions** - AI-generated hashtags
32. **AIUsageLogs** - AI usage tracking and costs

#### Analytics Service Tables (3)
33. **AnalyticsServiceRecords** - Detailed analytics records
34. **EngagementSnapshots** - Time-series engagement data
35. **PostPerformances** - Post performance rankings

#### Notification Service Tables (5)
36. **Notifications** - In-app notifications
37. **NotificationTemplates** - Email/push templates
38. **NotificationPreferences** - User notification settings
39. **EmailQueues** - Email sending queue
40. **DeviceTokens** - Push notification device tokens

#### Scheduler Service Tables (3)
41. **SchedulerScheduledJobs** - Background job queue
42. **SchedulerPublishLogs** - Publishing execution logs
43. **JobTemplates** - Reusable job templates

#### Webhook Service Tables (3)
44. **WebhookEvents** - Received webhook events
45. **WebhookSubscriptions** - Active webhook subscriptions
46. **WebhookProcessingLogs** - Webhook processing history

#### Developer API Service Tables (4)
47. **DeveloperApps** - Developer applications
48. **ApiKeys** - API keys (hashed)
49. **ApiUsages** - API usage tracking
50. **RateLimits** - Rate limiting records

#### Shared Tables (2)
51. **ApiRateLimits** - Generic API rate limits
52. **__EFMigrationsHistory** - EF Core migration history

### Database Modifications

All database changes are managed through **Entity Framework Core Migrations**:

**Location**: `NexusPost/src/Shared/Shared.Infrastructure/Migrations/`

**Key Migrations**:
1. `20251116071155_ConsolidatedDbContext` - Initial consolidated database
2. `20251116132233_UpdateTaskAssignmentForAgencyTasks` - Agency task support
3. `20251116140533_AddClientIdToUser` - Client user linking
4. `20251205080742_AddApiRateLimitsTable` - Rate limiting
5. `20251205121537_AddEmailQueueAndTenantBranding` - Email queue
6. `20251206012834_AddDeviceTokensTable` - Push notifications
7. `20251208054156_AddPublicIdAndThumbnailUrlToMediaAsset` - Media improvements

**How Migrations Work**:
- Migrations run automatically on application startup
- Located in `Program.cs` → `RunMigrationsAsync()`
- Retries up to 3 times with exponential backoff
- Handles Azure SQL connection issues gracefully

---

## RabbitMQ Messaging

### What is RabbitMQ?

RabbitMQ is a **message broker** that enables asynchronous communication between services. Instead of services calling each other directly, they publish events to queues, and other services consume those events.

### Why We Use RabbitMQ

1. **Decoupling**: Services don't need to know about each other
2. **Scalability**: Can process events in background
3. **Reliability**: Messages are persisted, won't be lost
4. **Flexibility**: Easy to add new event consumers

### How It Works

```
Service A (Publisher)
    │
    │ Publishes Event
    ▼
RabbitMQ Queue
    │
    │ Message Stored
    ▼
Service B (Consumer)
    │
    │ Consumes Event
    ▼
Background Processing
```

### Configuration

**Location**: `NexusPost/src/Shared/Shared.Messaging/`

**Files**:
- `RabbitMQPublisher.cs` - Publishes messages
- `RabbitMQConsumer.cs` - Consumes messages
- `ServiceCollectionExtensions.cs` - Configuration

**Connection Settings**:
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

### Event Flow Architecture

#### 1. Post Published Event

```
ContentService (Publishes)
    │
    │ POST /api/posts/{id}/publish
    │
    ├──► Creates PostPublishedEvent
    │
    └──► Publishes to "post.published" queue
         │
         ├──► NotificationService (Consumes)
         │    └──► Sends notification to user
         │
         └──► AnalyticsService (Consumes)
              └──► Creates analytics record
```

#### 2. Post Approved Event

```
ContentService (Publishes)
    │
    │ POST /api/approvals/review (Approve)
    │
    ├──► Creates PostApprovedEvent
    │
    └──► Publishes to "post.approved" queue
         │
         ├──► NotificationService (Consumes)
         │    └──► Notifies post creator
         │
         └──► SchedulerService (Consumes)
              └──► Schedules post for publishing
```

#### 3. Social Account Connected Event

```
SocialAccountService (Publishes)
    │
    │ OAuth Callback Success
    │
    ├──► Creates SocialAccountConnectedEvent
    │
    └──► Publishes to "socialaccount.connected" queue
         │
         └──► NotificationService (Consumes)
              └──► Welcomes user, sends connection confirmation
```

### Event Types

| Event | Published By | Consumed By | Queue Name |
|-------|-------------|-------------|------------|
| `PostPublishedEvent` | ContentService | NotificationService, AnalyticsService | `post.published` |
| `PostApprovedEvent` | ContentService | NotificationService, SchedulerService | `post.approved` |
| `PostCreatedEvent` | ContentService | (Future use) | `post.created` |
| `PostScheduledEvent` | ContentService | (Future use) | `post.scheduled` |
| `SocialAccountConnectedEvent` | SocialAccountService | NotificationService | `socialaccount.connected` |
| `SocialAccountDisconnectedEvent` | SocialAccountService | (Future use) | `socialaccount.disconnected` |
| `TokenExpiredEvent` | SocialAccountService | (Future use) | `token.expired` |
| `UserRegisteredEvent` | AuthService | (Future use) | `user.registered` |
| `UserLoginEvent` | AuthService | (Future use) | `user.login` |
| `UserLogoutEvent` | AuthService | (Future use) | `user.logout` |
| `WebhookReceivedEvent` | WebhookService | (Future use) | `webhook.received` |

### Message Publishing Example

```csharp
// In ContentService
await _messagePublisher.PublishAsync("post.published", new PostPublishedEvent
{
    PostId = post.Id,
    TenantId = post.TenantId,
    UserId = post.CreatedByUserId,
    PublishedAt = DateTime.UtcNow
});
```

### Message Consumption Example

```csharp
// In NotificationService (Background Service)
await _messageConsumer.StartConsumingAsync<PostPublishedEvent>(
    "post.published",
    async (eventData) =>
    {
        // Create notification
        await _notificationService.CreateNotificationAsync(...);
    }
);
```

### Background Services

These services run continuously and consume RabbitMQ events:

1. **NotificationService.RabbitMQConsumerService**
   - Consumes: `post.published`, `post.approved`, `socialaccount.connected`

2. **AnalyticsService.RabbitMQConsumerService**
   - Consumes: `post.published`

3. **SchedulerService.RabbitMQConsumerService**
   - Consumes: `post.approved`, `post.created.from.webhook`

4. **EmailQueueProcessor** (NotificationService)
   - Processes email queue (not RabbitMQ, but background service)

5. **JobProcessorService** (SchedulerService)
   - Processes scheduled jobs (not RabbitMQ, but background service)

6. **AnalyticsRefreshService** (AnalyticsService)
   - Periodically refreshes analytics data

---

## API Endpoints

### Total: 195 Endpoints

See detailed endpoint documentation in: `API_ENDPOINTS_REFERENCE.md`

### Quick Summary by Service

- **AuthService**: 32 endpoints (register, login, team management, admin)
- **ContentService**: 27 endpoints (posts, media, clients, approvals)
- **CollabService**: 21 endpoints (workspaces, teams, comments, tasks)
- **SocialAccountService**: 16 endpoints (OAuth, connections)
- **NotificationService**: 15 endpoints (notifications, emails, devices)
- **SchedulerService**: 12 endpoints (scheduling, publishing)
- **AIService**: 9 endpoints (content generation, sentiment, hashtags)
- **BillingService**: 9 endpoints (subscriptions, payments, invoices)
- **WebhookService**: 8 endpoints (webhook processing, subscriptions)
- **DeveloperAPIService**: 8 endpoints (API keys, apps, usage)
- **AnalyticsService**: 7 endpoints (analytics, performance)

---

## Feature Implementation Details

For detailed feature-by-feature documentation, see:
- `SERVICES_FEATURE_DOCUMENTATION.md` - Complete feature breakdown
- `RABBITMQ_DETAILED_GUIDE.md` - RabbitMQ deep dive
- `DATABASE_SCHEMA_DETAILED.md` - Complete database schema

---

## Next Steps

1. Read `SERVICES_FEATURE_DOCUMENTATION.md` for feature details
2. Read `RABBITMQ_DETAILED_GUIDE.md` for messaging architecture
3. Read `DATABASE_SCHEMA_DETAILED.md` for database structure
4. Read `API_ENDPOINTS_REFERENCE.md` for all endpoints

---

**Last Updated**: December 2024
**Version**: 1.0.0

