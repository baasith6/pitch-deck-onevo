# Onevo - Services Feature Documentation

## Complete Feature-by-Feature Breakdown

This document provides detailed information about every feature in NexusPost, how it's implemented, and how it's used in the application.

---

## Table of Contents

1. [Service 1: AuthService](#service-1-authservice)
2. [Service 2: ContentService](#service-2-contentservice)
3. [Service 3: SocialAccountService](#service-3-socialaccountservice)
4. [Service 4: SchedulerService](#service-4-schedulerservice)
5. [Service 5: AnalyticsService](#service-5-analyticsservice)
6. [Service 6: AIService](#service-6-aiservice)
7. [Service 7: NotificationService](#service-7-notificationservice)
8. [Service 8: CollabService](#service-8-collabservice)
9. [Service 9: BillingService](#service-9-billingservice)
10. [Service 10: WebhookService](#service-10-webhookservice)
11. [Service 11: DeveloperAPIService](#service-11-developerapiservice)

---

## Service 1: AuthService

### Purpose
Handles all authentication, authorization, user management, and tenant management.

### Controllers (3)
- **AuthController** (15 endpoints)
- **AdminController** (15 endpoints)
- **UserSettingsController** (2 endpoints)

### Features

#### 1. User Registration
**Endpoint**: `POST /api/auth/register`

**How It Works**:
1. User provides email, password, tenant name
2. System creates new Tenant (organization)
3. Creates User account linked to tenant
4. Hashes password using BCrypt
5. Generates JWT access token and refresh token
6. Publishes `UserRegisteredEvent` to RabbitMQ

**Implementation**:
- Uses `PasswordService` for secure password hashing
- Uses `JwtService` for token generation
- Creates tenant with type "Individual" or "Agency"
- Returns `AuthResponse` with tokens and user info

**Database Tables**:
- `Tenants` - New tenant record
- `Users` - New user record
- `RefreshTokens` - Refresh token stored

#### 2. User Login
**Endpoint**: `POST /api/auth/login`

**How It Works**:
1. User provides email and password
2. System validates credentials
3. Verifies password hash
4. Generates new JWT tokens
5. Stores refresh token in database
6. Returns tokens and user information

**Implementation**:
- Password verification using BCrypt
- JWT token includes: UserId, Email, Role, TenantId, TenantName, TenantType
- Refresh token expires in 7 days
- Access token expires in 1 hour

#### 3. Token Refresh
**Endpoint**: `POST /api/auth/refresh`

**How It Works**:
1. User sends refresh token
2. System validates refresh token
3. Checks if token is expired or revoked
4. Generates new access token
5. Optionally rotates refresh token

**Implementation**:
- Validates token signature and expiration
- Checks token in database (not revoked)
- Issues new access token without requiring new login

#### 4. Team Member Management
**Endpoints**:
- `POST /api/auth/team-member` - Create team member
- `GET /api/auth/team-members` - List team members
- `PUT /api/auth/team-member/{userId}` - Update team member
- `DELETE /api/auth/team-member/{userId}` - Delete team member

**How It Works**:
- Agencies can create team members
- Team members belong to same tenant
- Each team member gets their own user account
- Permissions managed through CollabService

**Use Case**: Agency creates team members to manage client accounts

#### 5. Client User Management
**Endpoints**:
- `POST /api/auth/client-user` - Create client user account
- `GET /api/auth/client-user/{clientId}` - Get client user
- `POST /api/auth/client-user/{clientId}/access` - Impersonate client

**How It Works**:
- Agencies create user accounts for their clients
- Client users have `ClientId` linked to `Clients` table
- Agency users can "impersonate" client to access their dashboard
- Client users see only their own data

**Use Case**: Agency manages multiple clients, each client has their own login

#### 6. Admin Functions
**Endpoints**: All under `/api/admin/*`

**Features**:
- View all users, tenants, posts (admin only)
- Create/update/delete tenants
- Create/update/delete users
- System-wide management

**Access Control**: Requires `SuperAdmin` role and `System` tenant type

#### 7. User Settings
**Endpoints**:
- `GET /api/usersettings` - Get user preferences
- `PUT /api/usersettings` - Update user preferences

**Settings Include**:
- Theme (light/dark)
- Timezone
- Date/time format
- Language
- Default platforms
- Notification preferences
- AI model preferences

---

## Service 2: ContentService

### Purpose
Manages all content creation, posts, media, clients, and approval workflows.

### Controllers (4)
- **PostsController** (12 endpoints)
- **MediaController** (5 endpoints)
- **ClientsController** (5 endpoints)
- **ApprovalsController** (5 endpoints)

### Features

#### 1. Post Creation
**Endpoint**: `POST /api/posts`

**How It Works**:
1. User creates post with content, media, target platforms
2. System validates permissions
3. Creates Post record with status "Draft"
4. Links media asset if provided
5. Creates PostTarget records for each selected platform
6. Publishes `PostCreatedEvent` to RabbitMQ

**Implementation Logic**:
```csharp
// 1. Check permissions
var permissions = await GetPermissionsAsync(tenantId, userId);
if (!permissions.CanCreatePosts) throw ForbiddenException;

// 2. Get or create client (for individual users)
Guid clientId = await EnsureClientExistsAsync(tenantId, userId);

// 3. Create post
var post = new Post {
    Content = request.Content,
    ClientId = clientId,
    Status = "Draft",
    CreatedByUserId = userId,
    TenantId = tenantId
};

// 4. Add media if provided
if (request.MediaId.HasValue) {
    post.MediaId = request.MediaId;
}

// 5. Create post targets for each platform
foreach (var socialAccountId in request.SocialAccountIds) {
    var target = new PostTarget {
        PostId = post.Id,
        SocialAccountId = socialAccountId,
        Platform = socialAccount.Platform
    };
}

// 6. Save to database
await _postRepository.AddAsync(post);
await _unitOfWork.SaveChangesAsync();

// 7. Publish event
await _messagePublisher.PublishAsync("post.created", event);
```

**Database Changes**:
- `Posts` table - New post record
- `PostTargets` table - One record per selected platform
- `PostVersions` table - Initial version created

#### 2. Post Scheduling
**Endpoint**: `POST /api/posts/{postId}/schedule`

**How It Works**:
1. User schedules post for future publishing
2. System creates ScheduledJob record
3. Post status changes to "Scheduled"
4. Background service (JobProcessorService) monitors scheduled jobs
5. When time arrives, job is processed and post is published

**Implementation Logic**:
```csharp
// 1. Validate scheduled time is in future
if (request.ScheduledAt <= DateTime.UtcNow) {
    throw ValidationException("Scheduled time must be in the future");
}

// 2. Update post status
post.Status = "Scheduled";
post.ScheduledAt = request.ScheduledAt;

// 3. Create scheduled job
var scheduledJob = new ScheduledJob {
    PostId = postId,
    ScheduledAt = request.ScheduledAt,
    Status = "Scheduled",
    TenantId = tenantId
};

// 4. Save
await _scheduledJobRepository.AddAsync(scheduledJob);
await _unitOfWork.SaveChangesAsync();

// 5. Publish event
await _messagePublisher.PublishAsync("post.scheduled", event);
```

**Background Processing**:
- `JobProcessorService` runs every 30 seconds
- Checks for jobs where `ScheduledAt <= Now` and `Status = "Scheduled"`
- Calls `SocialPublishingService` to publish post
- Updates job status to "Completed" or "Failed"

#### 3. Post Publishing (Immediate)
**Endpoint**: `POST /api/posts/{postId}/publish`

**How It Works**:
1. User clicks "Publish Now"
2. System validates post has targets (platforms selected)
3. Calls `SocialPublishingService` to publish to each platform
4. Updates post status to "Published"
5. Creates PublishLog records for each platform
6. Publishes `PostPublishedEvent` to RabbitMQ

**Implementation Logic**:
```csharp
// 1. Get post and validate
var post = await _postRepository.GetByIdAsync(postId);
if (post.Status == "Published") {
    throw ConflictException("Post already published");
}

// 2. Get post targets (platforms)
var targets = await _postTargetRepository.GetByPostIdAsync(postId);
if (!targets.Any()) {
    throw ValidationException("No platforms selected");
}

// 3. Publish to each platform
var results = new List<PublishResult>();
foreach (var target in targets) {
    var socialAccount = await GetSocialAccountAsync(target.SocialAccountId);
    
    var publishRequest = new PublishPostRequest {
        PostId = postId,
        SocialAccountId = socialAccount.Id,
        Platform = socialAccount.Platform,
        Content = post.Content,
        MediaUrl = post.Media?.Url
    };
    
    var result = await _socialPublishingService.PublishPostAsync(publishRequest);
    results.Add(result);
    
    // Create publish log
    var log = new PublishLog {
        PostId = postId,
        Platform = socialAccount.Platform,
        Status = result.Success ? "Success" : "Failed",
        ErrorMessage = result.ErrorMessage
    };
}

// 4. Update post status
post.Status = "Published";
post.PublishedAt = DateTime.UtcNow;
await _postRepository.UpdateAsync(post);

// 5. Publish event
await _messagePublisher.PublishAsync("post.published", new PostPublishedEvent {
    PostId = postId,
    TenantId = tenantId,
    UserId = userId,
    PublishedAt = DateTime.UtcNow
});
```

**What Happens Next**:
- `PostPublishedEvent` is consumed by:
  - **NotificationService**: Sends notification to user
  - **AnalyticsService**: Creates analytics record

#### 4. Post Versioning
**Feature**: Every time a post is edited, a new version is created

**How It Works**:
1. User edits post content
2. System creates new `PostVersion` record
3. Stores old content, new content, version number
4. Post content is updated to new version
5. Old versions are preserved for history

**Implementation**:
```csharp
// Get next version number
var nextVersion = await _postVersionRepository.GetNextVersionNumberAsync(postId);

// Create version record
var version = new PostVersion {
    PostId = postId,
    VersionNumber = nextVersion,
    Content = request.Content, // New content
    EditedByUserId = userId
};

// Update post
post.Content = request.Content;
await _postRepository.UpdateAsync(post);
await _postVersionRepository.AddAsync(version);
```

**Use Case**: Track post changes, rollback to previous versions

#### 5. Approval Workflow
**Endpoints**:
- `POST /api/approvals/request` - Request approval
- `POST /api/approvals/review` - Approve/reject
- `GET /api/approvals/pending` - Get pending approvals

**How It Works**:
1. User creates post and requests approval
2. System creates `ApprovalRequest` with status "Pending"
3. Post status changes to "PendingApproval"
4. Reviewer (team member with permission) reviews post
5. Reviewer approves or rejects
6. If approved:
   - Post status changes to "Approved"
   - Publishes `PostApprovedEvent` to RabbitMQ
   - SchedulerService consumes event and schedules post
7. If rejected:
   - Post status changes to "Draft"
   - Notification sent to creator

**Implementation Logic**:
```csharp
// Request approval
var approvalRequest = new ApprovalRequest {
    PostId = postId,
    RequestedByUserId = userId,
    ReviewerId = request.ReviewerId,
    Status = "Pending"
};

post.Status = "PendingApproval";
await _approvalRequestRepository.AddAsync(approvalRequest);

// Review approval
if (request.Approved) {
    approvalRequest.Status = "Approved";
    approvalRequest.ReviewedAt = DateTime.UtcNow;
    post.Status = "Approved";
    
    // Publish event
    await _messagePublisher.PublishAsync("post.approved", new PostApprovedEvent {
        PostId = postId,
        ApprovalRequestId = approvalRequest.Id
    });
} else {
    approvalRequest.Status = "Rejected";
    post.Status = "Draft";
}
```

**Event Flow**:
```
Post Approved
    │
    ▼
PostApprovedEvent → RabbitMQ
    │
    ├──► NotificationService
    │    └──► Notifies creator "Post approved!"
    │
    └──► SchedulerService
         └──► If post has ScheduledAt, creates scheduled job
```

#### 6. Media Management
**Endpoints**:
- `POST /api/media/upload` - Upload media file
- `GET /api/media/{mediaId}` - Get media
- `DELETE /api/media/{mediaId}` - Delete media

**How It Works**:
1. User uploads file (image, video)
2. System stores file (currently URL-based, file storage needed)
3. Creates `MediaAsset` record
4. Returns media ID for use in posts

**Current Implementation**:
- Media URLs are stored in database
- Actual file storage needs to be implemented (Azure Blob, AWS S3, or local)

**Future Enhancement**: 
- File upload to cloud storage
- Image processing/resizing
- Thumbnail generation

#### 7. Client Management (Agencies)
**Endpoints**:
- `GET /api/clients` - List clients
- `POST /api/clients` - Create client
- `PUT /api/clients/{clientId}` - Update client
- `DELETE /api/clients/{clientId}` - Delete client

**How It Works**:
- Agencies create client records
- Each client can have posts, social accounts
- Agency users can switch between clients
- Individual users have one auto-created client

**Database**:
- `Clients` table stores client information
- `Users` table has `ClientId` for client users
- Posts are linked to clients via `ClientId`

---

## Service 3: SocialAccountService

### Purpose
Manages OAuth connections to social media platforms (Facebook, Instagram, Twitter, LinkedIn, YouTube).

### Controllers (2)
- **SocialAccountController** (13 endpoints)
- **PostTargetsController** (3 endpoints)

### Features

#### 1. OAuth Connection Flow
**Endpoint**: `POST /api/socialaccount/connect`

**How It Works**:
1. User selects platform (Facebook, Instagram, etc.)
2. System generates OAuth state token
3. Stores state in `OAuthStates` table
4. Redirects user to platform's OAuth page
5. User authorizes on platform
6. Platform redirects back to `/api/socialaccount/callback`
7. System exchanges authorization code for access token
8. Stores encrypted tokens in `SocialAccounts` table
9. Publishes `SocialAccountConnectedEvent` to RabbitMQ

**Implementation Logic**:
```csharp
// 1. Generate OAuth state
var state = Guid.NewGuid().ToString();
var oauthState = new OAuthState {
    State = state,
    Platform = request.Platform,
    UserId = userId,
    TenantId = tenantId,
    ExpiresAt = DateTime.UtcNow.AddMinutes(10)
};

// 2. Build OAuth URL
var authUrl = BuildOAuthUrl(platform, state, redirectUri);

// 3. Store state
await _oauthStateRepository.AddAsync(oauthState);

// 4. Return URL to frontend
return authUrl; // User redirected to this URL
```

**OAuth Callback**:
```csharp
// 1. Validate state
var oauthState = await _oauthStateRepository.GetByStateAsync(state);
if (oauthState == null || oauthState.ExpiresAt < DateTime.UtcNow) {
    throw ValidationException("Invalid or expired OAuth state");
}

// 2. Exchange code for token
var tokenResponse = await ExchangeCodeForToken(platform, code);

// 3. Get account info from platform
var accountInfo = await GetAccountInfo(platform, tokenResponse.AccessToken);

// 4. Encrypt tokens
var encryptedAccessToken = _tokenEncryptionService.Encrypt(tokenResponse.AccessToken);
var encryptedRefreshToken = _tokenEncryptionService.Encrypt(tokenResponse.RefreshToken);

// 5. Create or update social account
var socialAccount = new SocialAccount {
    Platform = platform,
    PlatformUserId = accountInfo.Id,
    PlatformUsername = accountInfo.Username,
    DisplayName = accountInfo.Name,
    AccessToken = encryptedAccessToken,
    RefreshToken = encryptedRefreshToken,
    TokenExpiresAt = tokenResponse.ExpiresAt,
    IsActive = true
};

// 6. Save
await _socialAccountRepository.AddAsync(socialAccount);

// 7. Mark OAuth state as used
oauthState.IsUsed = true;

// 8. Publish event
await _messagePublisher.PublishAsync("socialaccount.connected", event);
```

**Supported Platforms**:
- Facebook (Pages, Groups)
- Instagram (Business accounts)
- Twitter/X (Tweets)
- LinkedIn (Company pages, personal)
- YouTube (Channels)

#### 2. Token Refresh
**How It Works**:
- OAuth tokens expire (usually 60 days)
- System automatically refreshes tokens before expiration
- Uses refresh token to get new access token
- Updates `SocialAccounts` table with new tokens

**Implementation**:
- Background service checks for expiring tokens
- Calls platform's refresh token endpoint
- Updates encrypted tokens in database

#### 3. Account Disconnection
**Endpoint**: `POST /api/socialaccount/disconnect`

**How It Works**:
1. User disconnects account
2. System revokes tokens on platform
3. Marks account as inactive
4. Soft deletes account record

#### 4. Post Targets
**Endpoint**: `POST /api/posttargets/get-targets`

**How It Works**:
- Returns list of connected social accounts
- Used when user selects platforms for a post
- Filters by active accounts only
- Groups by platform type

---

## Service 4: SchedulerService

### Purpose
Manages scheduled post publishing, background job processing, and automated publishing.

### Controllers (2)
- **SchedulerController** (7 endpoints)
- **PublishingController** (5 endpoints)

### Features

#### 1. Scheduled Job Creation
**How It Works**:
1. User schedules post via ContentService
2. ContentService creates ScheduledJob record
3. Job has status "Scheduled" and `ScheduledAt` time
4. Background service (`JobProcessorService`) monitors jobs
5. When `ScheduledAt` arrives, job is processed

**Database**:
- `SchedulerScheduledJobs` table stores jobs
- Fields: PostId, ScheduledAt, Status, RetryCount, MaxRetries

#### 2. Background Job Processing
**Service**: `JobProcessorService` (Background Service)

**How It Works**:
```csharp
// Runs every 30 seconds
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        // 1. Get pending jobs (ScheduledAt <= Now, Status = "Scheduled")
        var pendingJobs = await _scheduledJobRepository.GetPendingJobsAsync();
        
        foreach (var job in pendingJobs)
        {
            try
            {
                // 2. Update job status to "Processing"
                job.Status = "Processing";
                await _scheduledJobRepository.UpdateAsync(job);
                
                // 3. Get post
                var post = await _postRepository.GetByIdAsync(job.PostId);
                
                // 4. Get post targets (platforms)
                var targets = await _postTargetRepository.GetByPostIdAsync(post.Id);
                
                // 5. Publish to each platform
                foreach (var target in targets)
                {
                    var result = await _socialPublishingService.PublishPostAsync(...);
                    
                    // 6. Create publish log
                    var log = new PublishLog {
                        ScheduledJobId = job.Id,
                        PostId = post.Id,
                        Platform = target.Platform,
                        Status = result.Success ? "Success" : "Failed"
                    };
                }
                
                // 7. Update job status
                job.Status = "Completed";
            }
            catch (Exception ex)
            {
                // 8. Handle failure
                job.Status = "Failed";
                job.ErrorMessage = ex.Message;
                job.RetryCount++;
                
                // Retry if under max retries
                if (job.RetryCount < job.MaxRetries)
                {
                    job.Status = "Scheduled";
                    job.ScheduledAt = DateTime.UtcNow.AddMinutes(5); // Retry in 5 minutes
                }
            }
        }
        
        await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
    }
}
```

**Retry Logic**:
- Failed jobs are retried up to 3 times
- Retry delay: 5 minutes
- After max retries, job status = "Failed"

#### 3. Publishing to Social Platforms
**Service**: `SocialPublishingService`

**How It Works**:
1. Gets social account and access token
2. Validates token (checks expiration)
3. Refreshes token if needed
4. Calls platform-specific API
5. Creates publish log
6. Returns success/failure

**Platform-Specific Logic**:
- **Facebook**: Uses Graph API, posts to page
- **Instagram**: Uses Instagram Graph API
- **Twitter**: Uses Twitter API v2
- **LinkedIn**: Uses LinkedIn API
- **YouTube**: Uses YouTube Data API

**Error Handling**:
- Token expired → Refresh token and retry
- Rate limit → Queue for later
- Invalid content → Return error
- Network error → Retry with exponential backoff

#### 4. Event Consumption
**Service**: `RabbitMQConsumerService` (Background Service)

**Consumes Events**:
- `PostApprovedEvent` - When post is approved, schedule it if it has ScheduledAt
- `PostCreatedFromWebhookEvent` - When webhook creates post, schedule it

**Implementation**:
```csharp
await _messageConsumer.StartConsumingAsync<PostApprovedEvent>(
    "post.approved.queue",
    async (eventData) =>
    {
        var post = await GetPostAsync(eventData.PostId);
        
        // If post has ScheduledAt, create scheduled job
        if (post.ScheduledAt.HasValue)
        {
            await SchedulePostAsync(new SchedulePostRequest {
                PostId = post.Id,
                ScheduledAt = post.ScheduledAt.Value
            });
        }
    }
);
```

---

## Service 5: AnalyticsService

### Purpose
Tracks post performance, engagement metrics, and provides analytics insights.

### Controllers (1)
- **AnalyticsController** (7 endpoints)

### Features

#### 1. Analytics Record Creation
**How It Works**:
1. When post is published, `PostPublishedEvent` is published
2. AnalyticsService consumes event
3. Creates `AnalyticsRecord` for each platform
4. Initializes metrics to zero
5. Background service periodically refreshes metrics

**Implementation**:
```csharp
// Event handler
public async Task HandleAsync(PostPublishedEvent eventData)
{
    // Get post targets (platforms)
    var targets = await GetPostTargetsAsync(eventData.PostId);
    
    foreach (var target in targets)
    {
        var analyticsRecord = new AnalyticsRecord {
            PostId = eventData.PostId,
            TenantId = eventData.TenantId,
            Platform = target.Platform,
            PlatformPostId = target.PlatformPostId,
            Likes = 0,
            Comments = 0,
            Shares = 0,
            Views = 0,
            Status = "Active"
        };
        
        await _analyticsRecordRepository.AddAsync(analyticsRecord);
    }
}
```

#### 2. Metrics Tracking
**Metrics Tracked**:
- Likes/Reactions
- Comments
- Shares/Retweets
- Views/Impressions
- Clicks
- Reach
- Engagement Rate (calculated)

**Database**:
- `AnalyticsServiceRecords` - Main analytics data
- `EngagementSnapshots` - Time-series data (hourly/daily snapshots)
- `PostPerformances` - Aggregated performance rankings

#### 3. Analytics Refresh
**Service**: `AnalyticsRefreshService` (Background Service)

**How It Works**:
1. Runs every hour
2. Gets all active posts
3. Calls platform APIs to fetch latest metrics
4. Updates `AnalyticsRecord` with new data
5. Creates `EngagementSnapshot` for historical tracking

**Implementation**:
```csharp
// Get posts published in last 30 days
var posts = await GetRecentPublishedPostsAsync(days: 30);

foreach (var post in posts)
{
    var analyticsRecords = await GetAnalyticsRecordsAsync(post.Id);
    
    foreach (var record in analyticsRecords)
    {
        // Fetch from platform API
        var metrics = await FetchMetricsFromPlatform(
            record.Platform,
            record.PlatformPostId
        );
        
        // Update record
        record.Likes = metrics.Likes;
        record.Comments = metrics.Comments;
        record.Shares = metrics.Shares;
        record.Views = metrics.Views;
        record.EngagementRate = CalculateEngagementRate(metrics);
        
        // Create snapshot
        var snapshot = new EngagementSnapshot {
            AnalyticsRecordId = record.Id,
            SnapshotDate = DateTime.UtcNow,
            Likes = metrics.Likes,
            Comments = metrics.Comments,
            // ... other metrics
        };
    }
}
```

#### 4. Performance Rankings
**Feature**: Ranks posts by performance

**How It Works**:
- Calculates overall engagement rate
- Ranks posts within tenant
- Updates `PostPerformances` table
- Used for "Top Performing Posts" feature

---

## Service 6: AIService

### Purpose
Provides AI-powered content generation, sentiment analysis, and hashtag suggestions.

### Controllers (1)
- **AIController** (9 endpoints)

### Features

#### 1. Content Generation
**Endpoint**: `POST /api/ai/generate`

**How It Works**:
1. User provides prompt and generation type
2. System calls OpenAI GPT-4 or Anthropic Claude
3. AI generates content based on prompt
4. Stores generation in `AIGenerations` table
5. Tracks cost and usage
6. Returns generated content

**Generation Types**:
- `Content` - Full social media post
- `Caption` - Post caption only
- `Hashtag` - Hashtag suggestions
- `ContentImprovement` - Improve existing content
- `ImageDescription` - Describe image for accessibility

**Implementation**:
```csharp
// 1. Create generation record
var generation = new AIGeneration {
    GenerationType = request.GenerationType,
    Prompt = request.Prompt,
    Model = request.Model ?? "gpt-4",
    Status = "Pending"
};

// 2. Call AI API
var (content, success, error, tokens, cost) = await CallAIService(
    provider: "OpenAI",
    model: "gpt-4",
    prompt: BuildPrompt(request.GenerationType, request.Prompt)
);

// 3. Update generation
generation.GeneratedContent = content;
generation.Status = success ? "Completed" : "Failed";
generation.TokensUsed = tokens;
generation.Cost = cost;

// 4. Save
await _generationRepository.AddAsync(generation);
await _aiUsageLogRepository.AddAsync(new AIUsageLog {
    ServiceType = "ContentGeneration",
    TokensUsed = tokens,
    Cost = cost
});
```

**Cost Tracking**:
- Tracks tokens used
- Calculates cost based on model pricing
- Stores in `AIUsageLogs` table
- Used for billing/usage reporting

#### 2. Sentiment Analysis
**Endpoint**: `POST /api/ai/analyze-sentiment`

**How It Works**:
1. User provides content to analyze
2. System calls AI service for sentiment analysis
3. Returns sentiment (positive/negative/neutral), score, confidence
4. Stores analysis in `SentimentAnalyses` table

**Use Case**: Analyze post content before publishing to ensure positive sentiment

#### 3. Hashtag Suggestions
**Endpoint**: `POST /api/ai/suggest-hashtags`

**How It Works**:
1. User provides content
2. AI analyzes content and suggests relevant hashtags
3. Returns list of hashtags with relevance scores
4. Stores in `HashtagSuggestions` table

**Implementation**:
- Uses AI to understand content context
- Suggests platform-specific hashtags
- Considers trending hashtags
- Returns top 10-20 suggestions

#### 4. Additional AI Features
- **Generate Captions**: `POST /api/ai/generate-captions`
- **Best Time to Post**: `POST /api/ai/best-time-to-post` - AI suggests optimal posting times
- **Content Plan**: `POST /api/ai/generate-content-plan` - Creates content calendar
- **Image Generation**: `POST /api/ai/generate-image` - Generate images (DALL-E)
- **Image Editing**: `POST /api/ai/edit-image` - Edit images with AI

---

## Service 7: NotificationService

### Purpose
Manages all notifications (in-app, email, push) and notification preferences.

### Controllers (3)
- **NotificationsController** (7 endpoints)
- **EmailController** (4 endpoints)
- **DeviceTokensController** (4 endpoints)

### Features

#### 1. In-App Notifications
**Endpoint**: `POST /api/notifications`

**How It Works**:
1. System creates notification
2. Stores in `Notifications` table
3. Sends via SignalR to user's browser (real-time)
4. User sees notification in UI
5. User marks as read

**Notification Types**:
- Post published
- Post approved/rejected
- Social account connected
- Team member added
- Comment on post
- Task assigned

**Real-Time Delivery**:
- Uses SignalR Hub (`NotificationHub`)
- Frontend connects to `/notificationHub`
- Server pushes notifications instantly
- No page refresh needed

**Implementation**:
```csharp
// Create notification
var notification = new Notification {
    UserId = userId,
    Type = "PostPublished",
    Title = "Post Published",
    Message = "Your post has been published successfully",
    Link = $"/posts/{postId}",
    IsRead = false
};

await _notificationRepository.AddAsync(notification);

// Send via SignalR
await _hubContext.Clients.User(userId.ToString())
    .SendAsync("ReceiveNotification", notification);
```

#### 2. Email Notifications
**Service**: `EmailService`

**How It Works**:
1. System creates email notification
2. Adds to `EmailQueues` table
3. Background service (`EmailQueueProcessor`) processes queue
4. Sends email via SMTP (currently simulated)
5. Updates queue status

**Email Types**:
- Welcome email
- Password reset
- Post published confirmation
- Approval notifications
- Weekly digest

**Current Status**: Email service is simulated (logs to console)
**Future**: Integrate with SendGrid, AWS SES, or SMTP server

#### 3. Push Notifications
**Endpoints**:
- `POST /api/notifications/devices/register` - Register device token
- `GET /api/notifications/devices` - Get user's devices
- `DELETE /api/notifications/devices/{id}` - Unregister device

**How It Works**:
1. User's device registers for push notifications
2. System stores device token in `DeviceTokens` table
3. When notification is created, system sends push to device
4. Uses Firebase Cloud Messaging (FCM) or Apple Push Notification Service (APNS)

**Current Status**: Push service is simulated
**Future**: Integrate FCM and APNS

#### 4. Event Consumption
**Service**: `RabbitMQConsumerService` (Background Service)

**Consumes Events**:
- `PostPublishedEvent` → Creates notification "Post published successfully"
- `PostApprovedEvent` → Creates notification "Post approved"
- `SocialAccountConnectedEvent` → Creates notification "Account connected"

**Implementation**:
```csharp
await _messageConsumer.StartConsumingAsync<PostPublishedEvent>(
    "post.published",
    async (eventData) =>
    {
        // Create notification
        await _notificationService.CreateNotificationAsync(new CreateNotificationRequest {
            UserId = eventData.UserId,
            Type = "PostPublished",
            Title = "Post Published",
            Message = "Your post has been published"
        });
    }
);
```

---

## Service 8: CollabService

### Purpose
Manages team collaboration, workspaces, comments, tasks, and team member permissions.

### Controllers (5)
- **WorkspacesController** (3 endpoints)
- **TeamMembersController** (6 endpoints)
- **CommentsController** (2 endpoints)
- **CollaborationRequestsController** (2 endpoints)
- **TasksController** (8 endpoints)

### Features

#### 1. Workspace Management
**Endpoints**:
- `POST /api/workspaces` - Create workspace
- `GET /api/workspaces` - List workspaces
- `GET /api/workspaces/{id}` - Get workspace

**How It Works**:
- Workspaces are containers for team collaboration
- Each workspace has team members
- Workspaces contain posts, comments, tasks
- Used for organizing work by project or client

**Use Case**: Agency creates workspace for each client project

#### 2. Team Member Management
**Endpoints**:
- `POST /api/teammembers` - Add team member
- `GET /api/teammembers/workspace/{workspaceId}` - List team members
- `PATCH /api/teammembers/{id}/role` - Update role
- `PATCH /api/teammembers/{id}/permissions` - Update permissions

**Roles**:
- Owner
- Admin
- Editor
- Viewer

**Permissions**:
- CanCreatePosts
- CanEditPosts
- CanDeletePosts
- CanApprovePosts
- CanManageTeam
- CanViewAnalytics

**How It Works**:
1. Workspace owner adds team member
2. System creates `TeamMember` record
3. Assigns role and permissions
4. Team member can now access workspace

#### 3. Comments
**Endpoints**:
- `POST /api/comments` - Create comment
- `GET /api/comments` - Get comments

**How It Works**:
- Users can comment on posts
- Comments are threaded (reply to comments)
- Stored in `Comments` table
- Linked to workspace and post

**Use Case**: Team members discuss post content before publishing

#### 4. Task Management
**Endpoints**:
- `POST /api/tasks` - Create task
- `PUT /api/tasks/{id}` - Update task
- `PATCH /api/tasks/{id}/status` - Update status
- `DELETE /api/tasks/{id}` - Delete task
- `GET /api/tasks/assigned` - Get assigned tasks
- `GET /api/tasks/workspace/{id}` - Get workspace tasks

**How It Works**:
- Tasks are assignments for team members
- Tasks can be linked to posts, clients, workspaces
- Tasks have status: ToDo, InProgress, Completed, Cancelled
- Tasks have priority: Low, Medium, High, Urgent

**Use Case**: Assign task to team member to create post for client

#### 5. Collaboration Requests
**Endpoints**:
- `POST /api/collaborationrequests` - Create request
- `POST /api/collaborationrequests/{id}/respond` - Accept/reject

**How It Works**:
- User sends collaboration request to another user
- Request can be for workspace access, post collaboration
- Recipient accepts or rejects
- If accepted, user gains access

---

## Service 9: BillingService

### Purpose
Manages subscriptions, payments, invoices, and billing plans using Stripe.

### Controllers (5)
- **SubscriptionsController** (5 endpoints)
- **BillingPlansController** (1 endpoint)
- **PaymentMethodsController** (1 endpoint)
- **InvoicesController** (1 endpoint)
- **WebhooksController** (1 endpoint - Stripe webhooks)

### Features

#### 1. Subscription Plans
**Plans**:
- **Basic**: $29.99/month - 100 posts/month, 1 user
- **Pro**: $79.99/month - 500 posts/month, 5 users
- **Enterprise**: $199.99/month - Unlimited posts/users
- **Agency**: $49.99/month per account - Unlimited accounts

**How It Works**:
- Plans are seeded on application startup
- Stored in `BillingPlans` table
- Plans define features and limits
- Users select plan during subscription

#### 2. Subscription Creation
**Endpoint**: `POST /api/subscriptions`

**How It Works**:
1. User selects plan
2. System creates Stripe customer
3. Creates Stripe subscription
4. Stores subscription in database
5. Starts 14-day free trial
6. After trial, charges user

**Implementation**:
```csharp
// 1. Get billing plan
var plan = await _billingPlanRepository.GetByPlanIdAsync(request.PlanId);

// 2. Create Stripe customer
var customer = await _stripeService.CreateCustomerAsync(
    email: userEmail,
    name: tenantName
);

// 3. Create Stripe subscription
var subscription = await _stripeService.CreateSubscriptionAsync(
    customerId: customer.Id,
    priceId: plan.StripePriceIdMonthly,
    trialDays: 14
);

// 4. Store in database
var dbSubscription = new Subscription {
    TenantId = tenantId,
    UserId = userId,
    PlanId = plan.PlanId,
    Status = "Trial",
    StripeCustomerId = customer.Id,
    StripeSubscriptionId = subscription.Id,
    MonthlyPrice = plan.MonthlyPrice,
    TrialStartDate = DateTime.UtcNow,
    TrialEndDate = DateTime.UtcNow.AddDays(14)
};

await _subscriptionRepository.AddAsync(dbSubscription);
```

#### 3. Stripe Webhook Handling
**Endpoint**: `POST /api/billing/webhooks/stripe`

**How It Works**:
1. Stripe sends webhook events
2. System verifies webhook signature
3. Processes event:
   - `invoice.payment_succeeded` → Update subscription, create invoice
   - `invoice.payment_failed` → Mark subscription as past due
   - `customer.subscription.deleted` → Cancel subscription
   - `customer.subscription.updated` → Update subscription

**Events Handled**:
- Payment success
- Payment failure
- Subscription cancellation
- Subscription updates
- Trial ending

#### 4. Agency Pricing
**Special Feature**: Per-account pricing

**How It Works**:
- Agency plan charges $49.99 per client account
- Agency can add/remove accounts
- System updates subscription when account count changes
- Billed monthly based on active account count

**Endpoint**: `PUT /api/subscriptions/{id}/account-count`

---

## Service 10: WebhookService

### Purpose
Handles incoming webhooks from external platforms and content sources.

### Controllers (2)
- **WebhooksController** (4 endpoints)
- **WebhookSubscriptionsController** (4 endpoints)

### Features

#### 1. Webhook Processing
**Endpoint**: `POST /api/webhooks/{platform}`

**How It Works**:
1. External platform sends webhook (Facebook, WordPress, etc.)
2. System receives webhook payload
3. Verifies webhook signature
4. Processes webhook data
5. Creates `WebhookEvent` record
6. Publishes `WebhookReceivedEvent` to RabbitMQ

**Supported Platforms**:
- Facebook (page events)
- Instagram (media events)
- WordPress (new post published)
- Shopify (product updates)
- Custom webhooks

#### 2. Webhook Verification
**How It Works**:
- Platforms verify webhook endpoint before subscribing
- System returns challenge token
- Platform confirms webhook is valid

**Endpoint**: `GET /api/webhooks/{platform}`

#### 3. Content Webhooks
**Endpoint**: `POST /api/webhooks/content`

**How It Works**:
1. External system (WordPress, Shopify) sends webhook
2. System creates post automatically
3. Schedules post if needed
4. Publishes `PostCreatedFromWebhookEvent`

**Use Case**: WordPress blog post automatically creates social media post

---

## Service 11: DeveloperAPIService

### Purpose
Manages developer API access, API keys, rate limiting, and usage tracking.

### Controllers (3)
- **DeveloperAppsController** (3 endpoints)
- **ApiKeysController** (4 endpoints)
- **ApiUsageController** (1 endpoint)

### Features

#### 1. Developer App Management
**Endpoints**:
- `POST /api/developerapps` - Create app
- `GET /api/developerapps` - List apps
- `GET /api/developerapps/{id}` - Get app

**How It Works**:
- Developers create applications
- Each app can have multiple API keys
- Apps have scopes and permissions
- Used for third-party integrations

#### 2. API Key Management
**Endpoints**:
- `POST /api/apikeys` - Create API key
- `GET /api/apikeys/app/{appId}` - List keys
- `POST /api/apikeys/{id}/revoke` - Revoke key
- `POST /api/apikeys/validate` - Validate key

**How It Works**:
1. Developer creates API key
2. System generates secure key (e.g., `nex_abc123...`)
3. Hashes key and stores hash in database
4. Returns key to developer (shown only once)
5. Developer uses key in API requests

**Security**:
- Keys are hashed (never stored in plain text)
- Keys can be revoked
- Keys have expiration dates
- Keys have scopes (what they can access)

#### 3. Rate Limiting
**How It Works**:
- Tracks API calls per key
- Limits: 1000 requests/hour per key (configurable)
- Stores rate limit data in `RateLimits` table
- Returns 429 (Too Many Requests) when limit exceeded

**Implementation**:
```csharp
// Check rate limit
var rateLimit = await _rateLimitRepository.GetCurrentWindowAsync(
    apiKeyId: apiKeyId,
    windowType: "Hour"
);

if (rateLimit.RequestCount >= rateLimit.Limit)
{
    throw new RateLimitExceededException("Rate limit exceeded");
}

// Increment count
rateLimit.RequestCount++;
await _rateLimitRepository.UpdateAsync(rateLimit);
```

#### 4. API Usage Tracking
**Endpoint**: `GET /api/apiusage`

**Tracks**:
- Endpoint called
- Request method
- Response status
- Response time
- Timestamp
- API key used

**Use Case**: Monitor API usage, billing, analytics

---

## How Features Work Together

### Complete Post Publishing Flow

```
1. User Creates Post
   ContentService.CreatePostAsync()
   ├──► Creates Post (Draft)
   ├──► Creates PostTargets (platforms)
   └──► Publishes PostCreatedEvent

2. User Requests Approval (Optional)
   ContentService.RequestApprovalAsync()
   ├──► Creates ApprovalRequest
   └──► Post status = "PendingApproval"

3. Reviewer Approves
   ContentService.ReviewApprovalAsync()
   ├──► Updates ApprovalRequest
   ├──► Post status = "Approved"
   └──► Publishes PostApprovedEvent
       │
       ├──► NotificationService (Consumes)
       │    └──► Sends notification
       │
       └──► SchedulerService (Consumes)
            └──► If ScheduledAt exists, creates ScheduledJob

4. Background Service Processes Job
   JobProcessorService (runs every 30s)
   ├──► Finds jobs where ScheduledAt <= Now
   ├──► Calls SocialPublishingService
   │    └──► Publishes to each platform
   │         ├──► Facebook API
   │         ├──► Instagram API
   │         └──► Twitter API
   ├──► Creates PublishLog for each platform
   └──► Updates job status = "Completed"

5. Post Published Event
   ContentService.PublishPostAsync()
   └──► Publishes PostPublishedEvent
       │
       ├──► NotificationService (Consumes)
       │    └──► Creates notification "Post published!"
       │
       └──► AnalyticsService (Consumes)
            └──► Creates AnalyticsRecord
                └──► Background service refreshes metrics hourly
```

### Social Account Connection Flow

```
1. User Clicks "Connect Facebook"
   SocialAccountService.ConnectAccountAsync()
   ├──► Creates OAuthState
   └──► Returns OAuth URL

2. User Authorizes on Facebook
   Facebook OAuth Page
   └──► User grants permissions

3. Facebook Redirects Back
   SocialAccountService.HandleOAuthCallbackAsync()
   ├──► Validates OAuth state
   ├──► Exchanges code for token
   ├──► Gets account info from Facebook
   ├──► Encrypts tokens
   ├──► Creates SocialAccount record
   └──► Publishes SocialAccountConnectedEvent
       │
       └──► NotificationService (Consumes)
            └──► Sends "Account connected!" notification
```

---

## Database Relationships

### Key Relationships

1. **Tenant → Users** (One-to-Many)
   - One tenant has many users
   - Users belong to one tenant

2. **User → Client** (Many-to-One, for client users)
   - Client users belong to one client
   - Agency users can access multiple clients

3. **Client → Posts** (One-to-Many)
   - One client has many posts
   - Posts belong to one client

4. **Post → PostTargets** (One-to-Many)
   - One post can target multiple platforms
   - Each PostTarget links to one SocialAccount

5. **Post → PostVersions** (One-to-Many)
   - One post has many versions (edit history)

6. **Post → ApprovalRequests** (One-to-Many)
   - One post can have multiple approval requests

7. **Post → ScheduledJobs** (One-to-Many)
   - One post can have multiple scheduled jobs (rescheduling)

8. **Post → AnalyticsRecords** (One-to-Many)
   - One post has analytics for each platform

9. **SocialAccount → SocialAccountPages** (One-to-Many)
   - One account can have multiple pages (Facebook pages)

10. **Workspace → TeamMembers** (One-to-Many)
    - One workspace has many team members

---

## Summary

### Total Features by Category

- **Authentication & Authorization**: 7 features
- **Content Management**: 7 features
- **Social Media Integration**: 4 features
- **Scheduling & Automation**: 3 features
- **Analytics & Reporting**: 4 features
- **AI Features**: 9 features
- **Notifications**: 4 features
- **Collaboration**: 5 features
- **Billing & Payments**: 4 features
- **Webhooks**: 3 features
- **Developer API**: 4 features

**Total**: 54 distinct features

### Technology Highlights

- **195 API Endpoints** across 31 controllers
- **60+ Database Tables** in single shared database
- **11 Microservices** in modular monolith architecture
- **Event-Driven** communication via RabbitMQ
- **Real-Time** notifications via SignalR
- **Background Processing** for scheduled jobs
- **Multi-Tenant** architecture with tenant isolation
- **OAuth Integration** with 5+ social platforms
- **Stripe Integration** for payments
- **AI Integration** with OpenAI and Anthropic

---

**Last Updated**: December 2024
**Version**: 1.0.0

