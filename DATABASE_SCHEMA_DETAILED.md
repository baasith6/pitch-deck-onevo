# NexusPost - Complete Database Schema Documentation

## Table of Contents

1. [Database Overview](#database-overview)
2. [Schema by Service](#schema-by-service)
3. [Table Details](#table-details)
4. [Relationships](#relationships)
5. [Indexes](#indexes)
6. [Migrations](#migrations)
7. [Data Modifications](#data-modifications)

---

## Database Overview

### Database Name
`NexusPost`

### Database Type
SQL Server (Azure SQL or Local SQL Server)

### Architecture
**Single Shared Database** (Modular Monolith)
- All 11 services share the same database
- Single `NexusPostDbContext` manages all tables
- Simplifies migrations and data consistency

### Total Tables
**60+ Tables** across all services

### Connection String
```
Server={server};Database=NexusPost;User Id={user};Password={password};TrustServerCertificate=True;Encrypt={true/false};
```

---

## Schema by Service

### Auth Service (4 Tables)

#### 1. Tenants
**Purpose**: Multi-tenant organization data

**Columns**:
- `Id` (Guid, PK)
- `Name` (string, 200) - Organization name
- `TenantType` (string, 50) - "Individual", "Agency", "System"
- `LogoUrl` (string, 1000, nullable)
- `PrimaryColor` (string, 20, nullable)
- `SecondaryColor` (string, 20, nullable)
- `CompanyName` (string, 200, nullable)
- `SupportEmail` (string, 255, nullable)
- `SupportUrl` (string, 1000, nullable)
- `IsDeleted` (bool) - Soft delete
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantType`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `Users`

#### 2. Users
**Purpose**: User accounts (individuals, team members, client users)

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid, FK → Tenants)
- `ClientId` (Guid, nullable, FK → Clients) - For client users
- `Email` (string, 255, unique) - Login email
- `PasswordHash` (string) - BCrypt hash
- `Role` (string, 50) - "Owner", "Admin", "Editor", "Viewer", "TeamMember", "ClientUser"
- `CreatedByUserId` (Guid, nullable, FK → Users) - Who created this user
- `UpdatedByUserId` (Guid, nullable, FK → Users) - Who last updated
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `Email` (unique)
- `TenantId`
- `ClientId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Tenant`
- Many-to-One: `Client` (optional)
- One-to-Many: `RefreshTokens`
- One-to-One: `UserSettings`
- One-to-Many: `CreatedUsers` (self-reference)
- One-to-Many: `UpdatedUsers` (self-reference)

#### 3. RefreshTokens
**Purpose**: JWT refresh token storage

**Columns**:
- `Id` (Guid, PK)
- `UserId` (Guid, FK → Users)
- `Token` (string) - Refresh token value
- `ExpiresAt` (DateTime) - Token expiration
- `IsRevoked` (bool) - If token was revoked
- `RevokedAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `UserId`
- `Token`
- `ExpiresAt`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `User`

#### 4. UserSettings
**Purpose**: User preferences and settings

**Columns**:
- `Id` (Guid, PK)
- `UserId` (Guid, FK → Users, unique) - One-to-one with User
- `ThemeMode` (string, 10) - "light", "dark", "auto"
- `ColorTheme` (string, 20) - Theme color
- `Timezone` (string, 100) - User timezone
- `DateFormat` (string, 20) - Date format preference
- `TimeFormat` (string, 2) - "12" or "24"
- `Language` (string, 10) - "en", "es", etc.
- `DefaultPlatforms` (string, 1000, nullable) - JSON array
- `DefaultSchedulingTime` (string, 10, nullable) - "09:00"
- `NotificationFrequency` (string, 20) - "realtime", "hourly", "daily"
- `QuietHoursStart` (string, 10, nullable) - "22:00"
- `QuietHoursEnd` (string, 10, nullable) - "08:00"
- `DefaultAnalyticsDateRange` (string, 20) - "7d", "30d", "90d"
- `DefaultMetrics` (string, 1000, nullable) - JSON array
- `DefaultAIModel` (string, 50) - "gpt-4", "claude-3"
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `UserId` (unique)
- `IsDeleted` (query filter)

**Relationships**:
- One-to-One: `User`

---

### Social Account Service (4 Tables)

#### 5. SocialAccounts
**Purpose**: Connected social media accounts

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `Platform` (string, 50) - "Facebook", "Instagram", "Twitter", "LinkedIn", "YouTube"
- `PlatformUserId` (string, 200) - External platform user ID
- `PlatformUsername` (string, 200)
- `DisplayName` (string, 200)
- `ProfilePictureUrl` (string, 1000, nullable)
- `AccessToken` (string, 2000, nullable) - Encrypted
- `RefreshToken` (string, 2000, nullable) - Encrypted
- `TokenExpiresAt` (DateTime, nullable)
- `Scope` (string, 1000, nullable) - OAuth scopes
- `IsActive` (bool) - If account is active
- `LastConnectedAt` (DateTime, nullable)
- `LastDisconnectedAt` (DateTime, nullable)
- `ConnectionError` (string, 2000, nullable)
- `ConnectionAttempts` (int) - Retry count
- `LastTokenRefreshAt` (DateTime, nullable)
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `UserId`
- `Platform`
- `(Platform, PlatformUserId)` (unique)
- `(TenantId, UserId, Platform)`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `SocialAccountPages`
- One-to-Many: `SocialAccountTokens`
- One-to-Many: `PostTargets`

#### 6. SocialAccountPages
**Purpose**: Facebook pages, Instagram accounts linked to main account

**Columns**:
- `Id` (Guid, PK)
- `SocialAccountId` (Guid, FK → SocialAccounts)
- `PlatformPageId` (string, 200) - External page ID
- `PageName` (string, 200)
- `PageType` (string, 100) - "page", "group", "profile"
- `PageCategory` (string, 100, nullable)
- `PageDescription` (string, 1000, nullable)
- `PagePictureUrl` (string, 1000, nullable)
- `PageUrl` (string, 1000, nullable)
- `Permissions` (string, 2000, nullable) - JSON
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `SocialAccountId`
- `PlatformPageId`
- `(SocialAccountId, PlatformPageId)` (unique)
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `SocialAccount`

#### 7. SocialAccountTokens
**Purpose**: Additional OAuth tokens (long-lived tokens, page tokens)

**Columns**:
- `Id` (Guid, PK)
- `SocialAccountId` (Guid, FK → SocialAccounts)
- `TokenType` (string, 50) - "access", "refresh", "page", "long_lived"
- `TokenValue` (string, 2000) - Encrypted
- `ExpiresAt` (DateTime, nullable)
- `Scope` (string, 1000, nullable)
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `SocialAccountId`
- `TokenType`
- `(SocialAccountId, TokenType)`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `SocialAccount`

#### 8. OAuthStates
**Purpose**: OAuth flow state management (CSRF protection)

**Columns**:
- `Id` (Guid, PK)
- `State` (string, 200, unique) - OAuth state token
- `Platform` (string, 50) - Platform name
- `UserId` (Guid)
- `TenantId` (Guid)
- `RedirectUrl` (string, 1000, nullable)
- `FrontendRedirectUrl` (string, 1000, nullable)
- `Scope` (string, 1000, nullable)
- `ExpiresAt` (DateTime) - State expiration (10 minutes)
- `IsUsed` (bool) - If state was used
- `UsedAt` (DateTime, nullable)
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `State` (unique)
- `UserId`
- `TenantId`
- `ExpiresAt`
- `IsDeleted` (query filter)

**Purpose**: Prevents CSRF attacks in OAuth flow

---

### Content Service (9 Tables)

#### 9. Clients
**Purpose**: Client accounts (for agencies)

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `Name` (string, 200) - Client name
- `Status` (string, 50) - "Active", "Inactive", "Archived"
- `Industry` (string, 100, nullable)
- `Website` (string, 200, nullable)
- `PrimaryContactName` (string, 100, nullable)
- `PrimaryContactEmail` (string, 255, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `(TenantId, Name)` (unique)
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `Posts`
- One-to-Many: `Users` (client users)

#### 10. Posts
**Purpose**: Social media posts

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `ClientId` (Guid, FK → Clients)
- `CreatedByUserId` (Guid) - Post creator
- `Content` (string, 4000) - Post text content
- `MediaId` (Guid, nullable, FK → MediaAssets)
- `Status` (string, 50) - "Draft", "Scheduled", "Published", "PendingApproval", "Approved", "Rejected"
- `ScheduledAt` (DateTime, nullable) - When to publish
- `PublishedAt` (DateTime, nullable) - When published
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `ClientId`
- `Status`
- `ScheduledAt`
- `TenantId`
- `CreatedByUserId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Client`
- Many-to-One: `MediaAsset` (optional)
- One-to-Many: `PostTargets`
- One-to-Many: `PostVersions`
- One-to-Many: `ApprovalRequests`
- One-to-Many: `ScheduledJobs`
- One-to-Many: `PublishLogs`
- One-to-Many: `AnalyticsRecords`

#### 11. MediaAssets
**Purpose**: Uploaded media files (images, videos)

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UploadedByUserId` (Guid)
- `Url` (string, 1000) - Media URL
- `ThumbnailUrl` (string, 1000, nullable) - Thumbnail URL
- `PublicId` (string, 500, nullable) - Cloud storage public ID
- `FileName` (string, 500, nullable)
- `FileSize` (long, nullable) - Bytes
- `FileType` (string, 100) - "image/jpeg", "video/mp4", etc.
- `Width` (int, nullable) - Image width
- `Height` (int, nullable) - Image height
- `Duration` (int, nullable) - Video duration (seconds)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `UploadedByUserId`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `Posts`
- One-to-Many: `PostVersions`

**Note**: Currently stores URLs only. File upload needs to be implemented.

#### 12. PostVersions
**Purpose**: Post edit history

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, FK → Posts)
- `VersionNumber` (int) - Version number (1, 2, 3...)
- `EditedByUserId` (Guid) - Who edited
- `Content` (string, 4000) - Content at this version
- `MediaId` (Guid, nullable, FK → MediaAssets)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `PostId`
- `(PostId, VersionNumber)` (unique)
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`
- Many-to-One: `MediaAsset` (optional)

#### 13. PostTargets
**Purpose**: Target platforms for posts

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, FK → Posts)
- `SocialAccountId` (Guid, FK → SocialAccounts)
- `Platform` (string, 50) - "Facebook", "Instagram", etc.
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `PostId`
- `SocialAccountId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`
- Many-to-One: `SocialAccount`

**Use Case**: One post can target multiple platforms (Facebook + Instagram + Twitter)

#### 14. ApprovalRequests
**Purpose**: Post approval workflow

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, FK → Posts)
- `RequestedByUserId` (Guid) - Who requested approval
- `ReviewerId` (Guid) - Who should review
- `Status` (string, 50) - "Pending", "Approved", "Rejected"
- `Comment` (string, 1000, nullable) - Reviewer comment
- `RequestedAt` (DateTime)
- `ReviewedAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `PostId`
- `Status`
- `ReviewerId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`

#### 15. ContentScheduledJobs
**Purpose**: Scheduled post jobs (ContentService)

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, FK → Posts)
- `ScheduledAt` (DateTime) - When to publish
- `Status` (string, 50) - "Scheduled", "Processing", "Completed", "Failed"
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `PostId`
- `ScheduledAt`
- `Status`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`

**Note**: Separate from `SchedulerScheduledJobs` (SchedulerService)

#### 16. ContentPublishLogs
**Purpose**: Publishing history (ContentService)

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, FK → Posts)
- `Platform` (string, 50)
- `Result` (string, 50) - "Success", "Failed"
- `ErrorMessage` (string, 2000, nullable)
- `PublishedAt` (DateTime)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `PostId`
- `Platform`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`

#### 17. ContentAnalyticsRecords
**Purpose**: Basic analytics (ContentService)

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, FK → Posts)
- `Platform` (string, 50)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `PostId`
- `Platform`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`

**Note**: Separate from `AnalyticsServiceRecords` (AnalyticsService)

---

### Billing Service (5 Tables)

#### 18. BillingPlans
**Purpose**: Subscription plans

**Columns**:
- `Id` (Guid, PK)
- `PlanId` (string, 100, unique) - "Basic", "Pro", "Enterprise", "Agency"
- `Name` (string, 200)
- `Description` (string, 1000, nullable)
- `SubscriptionType` (string, 50) - "User", "Agency"
- `MonthlyPrice` (decimal, 18,2)
- `AnnualPrice` (decimal, 18,2, nullable)
- `SetupFee` (decimal, 18,2, nullable)
- `AccountLimit` (int, nullable) - For agency plans
- `PostLimit` (int, nullable) - Posts per month
- `UserLimit` (int, nullable) - Team members
- `HasAdvancedAnalytics` (bool)
- `HasPrioritySupport` (bool)
- `HasCustomBranding` (bool)
- `StripePriceIdMonthly` (string, 200, nullable)
- `StripePriceIdAnnual` (string, 200, nullable)
- `StripeProductId` (string, 200, nullable)
- `Features` (string, 4000, nullable) - JSON array
- `IsActive` (bool)
- `DisplayOrder` (int)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `PlanId` (unique)
- `SubscriptionType`
- `IsActive`
- `IsDeleted` (query filter)

**Seeded Data**:
- Basic: $29.99/month
- Pro: $79.99/month
- Enterprise: $199.99/month
- Agency: $49.99/month per account

#### 19. Subscriptions
**Purpose**: Active subscriptions

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid, unique) - One subscription per tenant
- `UserId` (Guid) - Subscription owner
- `SubscriptionType` (string, 50) - "User", "Agency"
- `PlanId` (string, 100) - "Basic", "Pro", etc.
- `Status` (string, 50) - "Trial", "Active", "Cancelled", "Expired", "PastDue"
- `TrialStartDate` (DateTime, nullable)
- `TrialEndDate` (DateTime, nullable)
- `TrialCancelled` (bool)
- `TrialCancelledAt` (DateTime, nullable)
- `CurrentPeriodStart` (DateTime, nullable)
- `CurrentPeriodEnd` (DateTime, nullable)
- `CancelAtPeriodEnd` (DateTime, nullable)
- `CancelledAt` (DateTime, nullable)
- `StripeSubscriptionId` (string, 200, nullable)
- `StripeCustomerId` (string, 200, nullable)
- `StripePriceId` (string, 200, nullable)
- `StripePaymentMethodId` (string, 200, nullable)
- `MonthlyPrice` (decimal, 18,2)
- `SetupFee` (decimal, 18,2, nullable)
- `AccountLimit` (int, nullable) - For agency
- `CurrentAccountCount` (int, nullable) - For agency
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId` (unique)
- `UserId`
- `StripeSubscriptionId`
- `StripeCustomerId`
- `Status`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `Invoices`

#### 20. PaymentMethods
**Purpose**: Stripe payment methods

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `Type` (string, 50) - "card", "bank_account"
- `Last4` (string, 4, nullable) - Last 4 digits
- `Brand` (string, 50, nullable) - "visa", "mastercard"
- `Country` (string, 10, nullable)
- `StripePaymentMethodId` (string, 200) - Stripe ID
- `StripeCustomerId` (string, 200, nullable)
- `IsDefault` (bool)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `UserId`
- `StripePaymentMethodId`
- `StripeCustomerId`
- `IsDeleted` (query filter)

#### 21. Invoices
**Purpose**: Billing invoices

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `SubscriptionId` (Guid, nullable, FK → Subscriptions)
- `InvoiceNumber` (string, 100, unique)
- `Amount` (decimal, 18,2)
- `Tax` (decimal, 18,2, nullable)
- `TotalAmount` (decimal, 18,2)
- `Currency` (string, 10) - "USD"
- `Status` (string, 50) - "Draft", "Open", "Paid", "Void", "Uncollectible"
- `StripeInvoiceId` (string, 200, nullable)
- `StripeChargeId` (string, 200, nullable)
- `StripePaymentIntentId` (string, 200, nullable)
- `LineItems` (string, 8000, nullable) - JSON
- `Description` (string, 1000, nullable)
- `DueDate` (DateTime, nullable)
- `PaidAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `UserId`
- `SubscriptionId`
- `InvoiceNumber` (unique)
- `StripeInvoiceId`
- `Status`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Subscription` (optional)

#### 22. TrialUsages
**Purpose**: Free trial tracking

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `SubscriptionId` (Guid, FK → Subscriptions)
- `TrialStartDate` (DateTime)
- `TrialEndDate` (DateTime)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `TenantId`
- `UserId`
- `SubscriptionId`
- `TrialEndDate`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Subscription`

---

### Collab Service (6 Tables)

#### 23. Workspaces
**Purpose**: Team collaboration workspaces

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `CreatedBy` (Guid) - Creator user ID
- `Name` (string, 200)
- `Description` (string, 1000, nullable)
- `Settings` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `CreatedBy`
- `Name`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `TeamMembers`
- One-to-Many: `Comments`
- One-to-Many: `Tasks`
- One-to-Many: `CollaborationRequests`
- One-to-Many: `ActivityLogs`

#### 24. TeamMembers
**Purpose**: Workspace team members

**Columns**:
- `Id` (Guid, PK)
- `WorkspaceId` (Guid, FK → Workspaces)
- `TenantId` (Guid)
- `UserId` (Guid)
- `Role` (string, 50) - "Owner", "Admin", "Editor", "Viewer"
- `Permissions` (string, 2000, nullable) - JSON
- `InvitedByUserId` (Guid, nullable)
- `InvitedAt` (DateTime, nullable)
- `JoinedAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `(WorkspaceId, UserId)` (unique)
- `TenantId`
- `UserId`
- `WorkspaceId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Workspace`

#### 25. Comments
**Purpose**: Comments on posts, workspaces

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `WorkspaceId` (Guid, nullable, FK → Workspaces)
- `UserId` (Guid) - Comment author
- `RelatedEntityId` (Guid) - Post ID, Task ID, etc.
- `RelatedEntityType` (string, 100) - "Post", "Task"
- `ParentCommentId` (Guid, nullable, FK → Comments) - For threaded comments
- `Content` (string, 2000)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `WorkspaceId`
- `UserId`
- `(RelatedEntityId, RelatedEntityType)`
- `ParentCommentId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Workspace` (optional)
- Many-to-One: `Comment` (self-reference, for replies)

#### 26. Tasks
**Purpose**: Task assignments

**Columns**:
- `Id` (Guid, PK)
- `WorkspaceId` (Guid, nullable, FK → Workspaces)
- `TenantId` (Guid)
- `ClientId` (Guid, nullable, FK → Clients) - For agency tasks
- `AssignedToUserId` (Guid, nullable) - Task assignee
- `CreatedByUserId` (Guid) - Task creator
- `Title` (string, 200)
- `Description` (string, 4000, nullable)
- `Status` (string, 50) - "ToDo", "InProgress", "Completed", "Cancelled"
- `Priority` (string, 50) - "Low", "Medium", "High", "Urgent"
- `DueDate` (DateTime, nullable)
- `CompletedAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `WorkspaceId`
- `TenantId`
- `ClientId`
- `AssignedToUserId`
- `Status`
- `Priority`
- `DueDate`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Workspace` (optional)
- Many-to-One: `Client` (optional)

#### 27. CollaborationRequests
**Purpose**: Collaboration invites

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `WorkspaceId` (Guid, nullable, FK → Workspaces)
- `FromUserId` (Guid) - Who sent request
- `ToUserId` (Guid) - Who receives request
- `RequestType` (string, 50) - "WorkspaceInvite", "PostCollaboration"
- `Message` (string, 1000, nullable)
- `Status` (string, 50) - "Pending", "Accepted", "Rejected"
- `RelatedEntityId` (Guid, nullable)
- `RelatedEntityType` (string, 100, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `WorkspaceId`
- `FromUserId`
- `ToUserId`
- `Status`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Workspace` (optional)

#### 28. ActivityLogs
**Purpose**: Activity tracking

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `WorkspaceId` (Guid, nullable, FK → Workspaces)
- `UserId` (Guid) - Who performed action
- `ActivityType` (string, 100) - "PostCreated", "PostPublished", "CommentAdded"
- `Description` (string, 500)
- `RelatedEntityId` (Guid, nullable)
- `RelatedEntityType` (string, 100, nullable)
- `Metadata` (string, 4000, nullable) - JSON
- `ActivityDate` (DateTime)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `TenantId`
- `WorkspaceId`
- `UserId`
- `ActivityType`
- `ActivityDate`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Workspace` (optional)

---

### AI Service (4 Tables)

#### 29. AIGenerations
**Purpose**: AI-generated content

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `GenerationType` (string, 100) - "Content", "Caption", "Hashtag", "ContentImprovement"
- `Prompt` (string, 2000, nullable)
- `GeneratedContent` (string, 8000, nullable)
- `Model` (string, 100) - "gpt-4", "claude-3"
- `RelatedEntityId` (Guid, nullable) - Post ID, etc.
- `RelatedEntityType` (string, 100, nullable)
- `Status` (string, 50) - "Pending", "Completed", "Failed"
- `TokensUsed` (int, nullable)
- `Cost` (decimal, 18,4, nullable)
- `ErrorMessage` (string, 2000, nullable)
- `Settings` (string, 4000, nullable) - JSON
- `GeneratedAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `UserId`
- `GenerationType`
- `(RelatedEntityId, RelatedEntityType)`
- `GeneratedAt`
- `IsDeleted` (query filter)

#### 30. SentimentAnalyses
**Purpose**: Sentiment analysis results

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `RelatedEntityId` (Guid) - Post ID, etc.
- `RelatedEntityType` (string, 100)
- `Content` (string, 10000) - Analyzed content
- `Sentiment` (string, 50) - "Positive", "Negative", "Neutral"
- `SentimentScore` (decimal, 18,4) - -1.0 to 1.0
- `Confidence` (decimal, 18,4) - 0.0 to 1.0
- `Emotions` (string, 1000, nullable) - JSON array
- `Keywords` (string, 2000, nullable) - JSON array
- `Summary` (string, 2000, nullable)
- `Model` (string, 100)
- `AnalyzedAt` (DateTime)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `TenantId`
- `UserId`
- `(RelatedEntityId, RelatedEntityType)`
- `Sentiment`
- `AnalyzedAt`
- `IsDeleted` (query filter)

#### 31. HashtagSuggestions
**Purpose**: AI-generated hashtag suggestions

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `RelatedEntityId` (Guid, nullable) - Post ID
- `Content` (string, 5000, nullable) - Content analyzed
- `Hashtags` (string, 2000) - Comma-separated hashtags
- `HashtagsJson` (string, 4000, nullable) - JSON array with scores
- `Model` (string, 100)
- `SuggestedAt` (DateTime)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `TenantId`
- `UserId`
- `RelatedEntityId`
- `SuggestedAt`
- `IsDeleted` (query filter)

#### 32. AIUsageLogs
**Purpose**: AI usage tracking and cost

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid)
- `ServiceType` (string, 100) - "ContentGeneration", "SentimentAnalysis", "HashtagSuggestion"
- `Model` (string, 100)
- `Operation` (string, 200) - Operation name
- `TokensUsed` (int, nullable)
- `Cost` (decimal, 18,4, nullable)
- `Success` (bool)
- `ErrorMessage` (string, 2000, nullable)
- `RequestData` (string, 8000, nullable) - JSON
- `ResponseData` (string, 8000, nullable) - JSON
- `UsedAt` (DateTime)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `TenantId`
- `UserId`
- `ServiceType`
- `UsedAt`
- `IsDeleted` (query filter)

---

### Analytics Service (3 Tables)

#### 33. AnalyticsServiceRecords
**Purpose**: Detailed analytics records

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `PostId` (Guid, FK → Posts)
- `UserId` (Guid, nullable) - Post creator
- `Platform` (string, 50) - "Facebook", "Instagram", etc.
- `PlatformPostId` (string, 200, nullable) - External post ID
- `PlatformUrl` (string, 1000, nullable) - Link to post
- `Likes` (int) - Engagement metrics
- `Comments` (int)
- `Shares` (int)
- `Retweets` (int) - For Twitter
- `Saves` (int) - For Instagram
- `Views` (int)
- `Clicks` (int)
- `Impressions` (int)
- `Reach` (int)
- `EngagementRate` (decimal, 18,4) - Calculated
- `FirstEngagementAt` (DateTime, nullable)
- `LastEngagementAt` (DateTime, nullable)
- `PublishedAt` (DateTime, nullable)
- `AdditionalMetrics` (string, 4000, nullable) - JSON
- `Status` (string, 50) - "Active", "Archived"
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `PostId`
- `TenantId`
- `UserId`
- `(PostId, Platform)` (unique)
- `PlatformPostId`
- `PublishedAt`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post`
- One-to-Many: `EngagementSnapshots`

#### 34. EngagementSnapshots
**Purpose**: Time-series engagement data

**Columns**:
- `Id` (Guid, PK)
- `AnalyticsRecordId` (Guid, FK → AnalyticsServiceRecords)
- `SnapshotDate` (DateTime) - When snapshot was taken
- `Likes` (int)
- `Comments` (int)
- `Shares` (int)
- `Retweets` (int)
- `Saves` (int)
- `Views` (int)
- `Clicks` (int)
- `Impressions` (int)
- `Reach` (int)
- `EngagementRate` (decimal, 18,4)
- `TotalEngagements` (int) - Sum of all engagements
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)

**Indexes**:
- `AnalyticsRecordId`
- `SnapshotDate`
- `(AnalyticsRecordId, SnapshotDate)` (unique)
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `AnalyticsRecord`

**Purpose**: Track engagement over time (hourly/daily snapshots)

#### 35. PostPerformances
**Purpose**: Post performance rankings

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, unique) - One performance record per post
- `TenantId` (Guid)
- `UserId` (Guid, nullable)
- `PerformanceCategory` (string, 50) - "Top", "Average", "Low"
- `Rank` (int, nullable) - Ranking within tenant
- `OverallEngagementRate` (decimal, 18,4)
- `ClickThroughRate` (decimal, 18,4)
- `ShareRate` (decimal, 18,4)
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `PostId` (unique)
- `TenantId`
- `UserId`
- `Rank`
- `PerformanceCategory`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-One: `Post`

---

### Notification Service (5 Tables)

#### 36. Notifications
**Purpose**: In-app notifications

**Columns**:
- `Id` (Guid, PK)
- `UserId` (Guid) - Notification recipient
- `TenantId` (Guid)
- `Type` (string, 100) - "PostPublished", "PostApproved", "AccountConnected"
- `Title` (string, 500)
- `Message` (string, 2000)
- `Link` (string, 1000, nullable) - URL to related content
- `Icon` (string, 100, nullable)
- `ImageUrl` (string, 1000, nullable)
- `Priority` (string, 50) - "Low", "Normal", "High", "Urgent"
- `Category` (string, 100, nullable) - "Post", "Account", "Team"
- `RelatedEntityId` (Guid, nullable)
- `RelatedEntityType` (string, 100, nullable)
- `IsRead` (bool)
- `ReadAt` (DateTime, nullable)
- `SendError` (string, 2000, nullable)
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `UserId`
- `TenantId`
- `(UserId, IsRead)`
- `(UserId, Type)`
- `CreatedAt`
- `IsDeleted` (query filter)

**Relationships**:
- None (standalone)

#### 37. NotificationTemplates
**Purpose**: Email/push notification templates

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid, nullable) - Null for system templates
- `Type` (string, 100) - "PostPublished", "Welcome", etc.
- `Name` (string, 200)
- `Subject` (string, 500) - Email subject
- `Body` (string, 4000) - Email body (HTML)
- `DefaultPriority` (string, 50)
- `DefaultCategory` (string, 100, nullable)
- `Language` (string, 10) - "en", "es", etc.
- `IsActive` (bool)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `(Type, TenantId, Language)`
- `TenantId`
- `IsDeleted` (query filter)

#### 38. NotificationPreferences
**Purpose**: User notification preferences

**Columns**:
- `Id` (Guid, PK)
- `UserId` (Guid)
- `TenantId` (Guid)
- `NotificationType` (string, 100) - "PostPublished", "PostApproved", etc.
- `Frequency` (string, 50) - "realtime", "hourly", "daily", "weekly", "never"
- `Channels` (string, 500, nullable) - JSON array: ["in-app", "email", "push"]
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `(UserId, NotificationType)` (unique)
- `UserId`
- `TenantId`
- `IsDeleted` (query filter)

#### 39. EmailQueues
**Purpose**: Email sending queue

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid, nullable)
- `UserId` (Guid, nullable)
- `ToEmail` (string, 500)
- `ToName` (string, 200, nullable)
- `Subject` (string, 500)
- `HtmlContent` (string) - HTML email body
- `PlainTextContent` (string, nullable)
- `TemplateType` (string, 100, nullable)
- `Status` (string, 50) - "Pending", "Sending", "Sent", "Failed"
- `Priority` (string, 50) - "Low", "Normal", "High"
- `ScheduledFor` (DateTime, nullable) - When to send
- `SentAt` (DateTime, nullable)
- `ErrorMessage` (string, 2000, nullable)
- `RetryCount` (int)
- `MaxRetries` (int)
- `Metadata` (string, 4000, nullable) - JSON
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `Status`
- `(Status, ScheduledFor)`
- `TenantId`
- `UserId`
- `CreatedAt`
- `IsDeleted` (query filter)

**Processing**: Background service `EmailQueueProcessor` processes queue

#### 40. DeviceTokens
**Purpose**: Push notification device tokens

**Columns**:
- `Id` (Guid, PK)
- `UserId` (Guid)
- `TenantId` (Guid, nullable)
- `Token` (string, 2000) - FCM/APNS token
- `Platform` (string, 50) - "android", "ios", "web"
- `DeviceId` (string, 200, nullable) - Device identifier
- `UserAgent` (string, 500, nullable)
- `AppVersion` (string, 50, nullable)
- `Metadata` (string, 4000, nullable) - JSON
- `IsActive` (bool)
- `LastUsedAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `UserId`
- `TenantId`
- `Token`
- `(UserId, IsActive)`
- `(DeviceId, UserId)`
- `IsDeleted` (query filter)

---

### Scheduler Service (3 Tables)

#### 41. SchedulerScheduledJobs
**Purpose**: Background scheduled jobs

**Columns**:
- `Id` (Guid, PK)
- `PostId` (Guid, nullable, FK → Posts)
- `TenantId` (Guid)
- `CreatedByUserId` (Guid)
- `ScheduledAt` (DateTime) - When to execute job
- `ExecutedAt` (DateTime, nullable) - When job was executed
- `Status` (string, 50) - "Scheduled", "Executing", "Completed", "Failed", "Cancelled"
- `ErrorMessage` (string, 2000, nullable)
- `RetryCount` (int) - Default: 0
- `MaxRetries` (int) - Default: 3
- `JobData` (string, 4000, nullable) - JSON data for job execution
- `Result` (string, 4000, nullable) - JSON result data
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `PostId`
- `TenantId`
- `ScheduledAt`
- `Status`
- `(Status, ScheduledAt)` - Composite for job processing queries
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `Post` (optional)
- One-to-Many: `PublishLogs`

**Note**: Separate from `ContentScheduledJobs` (ContentService). This table is used by the SchedulerService for background job processing.

#### 42. SchedulerPublishLogs
**Purpose**: Publishing execution logs (SchedulerService)

**Columns**:
- `Id` (Guid, PK)
- `ScheduledJobId` (Guid, nullable, FK → SchedulerScheduledJobs)
- `PostId` (Guid, FK → Posts)
- `Platform` (string, 50) - "Facebook", "Instagram", etc.
- `SocialAccountId` (string, 100) - Social account identifier
- `Status` (string, 50) - "Success", "Failed", "Pending"
- `ErrorMessage` (string, 2000, nullable)
- `ResponseData` (string, 4000, nullable) - JSON response from social platform
- `AttemptedAt` (DateTime) - When publish was attempted
- `RetryAttempt` (int) - Retry attempt number
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `ScheduledJobId`
- `PostId`
- `Platform`
- `Status`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `ScheduledJob` (optional)
- Many-to-One: `Post`

**Note**: Separate from `ContentPublishLogs` (ContentService)

#### 43. JobTemplates
**Purpose**: Reusable job templates

**Columns**:
- `Id` (Guid, PK)
- `Name` (string, 200)
- `Description` (string, 1000, nullable)
- `JobType` (string, 100) - "PostPublish", "AnalyticsSync", etc.
- `TemplateData` (string, 4000, nullable) - JSON template configuration
- `IsActive` (bool) - Default: true
- `Priority` (int) - Higher number = higher priority, Default: 0
- `Timeout` (TimeSpan, nullable) - Job timeout duration
- `MaxRetries` (int) - Default: 3
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `JobType`
- `IsActive`
- `IsDeleted` (query filter)

**Relationships**:
- None (standalone template)

---

### Webhook Service (3 Tables)

#### 44. WebhookEvents
**Purpose**: Received webhook events from external platforms

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `Platform` (string, 50) - "Facebook", "Instagram", "Twitter", "WordPress", "Shopify"
- `EventType` (string, 100) - "post_engagement", "comment", "like", "new_post"
- `SubscriptionId` (string, 200, nullable) - Webhook subscription ID
- `Payload` (string, 8000) - JSON payload from the platform
- `Signature` (string, 500, nullable) - Webhook signature for verification
- `Headers` (string, 2000, nullable) - HTTP headers (stored as JSON string)
- `ProcessedData` (string, 8000, nullable) - Processed/parsed data (JSON)
- `Status` (string, 50) - "Received", "Processing", "Processed", "Failed", "Ignored"
- `ErrorMessage` (string, 2000, nullable)
- `RelatedPostId` (Guid, nullable) - Related post if applicable
- `RelatedSocialAccountId` (Guid, nullable) - Related social account
- `RelatedEntityType` (string, 100, nullable)
- `RetryCount` (int) - Default: 0
- `ProcessedAt` (DateTime, nullable)
- `ReceivedAt` (DateTime) - When webhook was received
- `Metadata` (string, 4000, nullable) - JSON string for additional data
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `Platform`
- `(Platform, EventType)` - Composite for filtering
- `ReceivedAt`
- `Status`
- `RelatedPostId`
- `RelatedSocialAccountId`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `WebhookProcessingLogs`

#### 45. WebhookSubscriptions
**Purpose**: Active webhook subscriptions

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `SocialAccountId` (Guid, nullable, FK → SocialAccounts) - Optional: specific social account
- `Platform` (string, 50) - "Facebook", "Instagram", etc.
- `SubscriptionId` (string, 200) - Platform's subscription ID
- `CallbackUrl` (string, 1000) - Our webhook endpoint URL
- `VerifyToken` (string, 200) - Token for webhook verification
- `Secret` (string, 500, nullable) - Secret for signature verification (encrypted)
- `WebhookToken` (string, nullable) - Token for authenticating external websites (WordPress, Shopify, etc.)
- `EventTypes` (string, 500, nullable) - Comma-separated list of event types
- `IsActive` (bool) - Default: true
- `SubscribedAt` (DateTime, nullable)
- `ExpiresAt` (DateTime, nullable)
- `IsVerified` (bool) - Default: false
- `VerifiedAt` (DateTime, nullable)
- `VerificationError` (string, 2000, nullable)
- `TotalEventsReceived` (int) - Default: 0
- `SuccessfulEvents` (int) - Default: 0
- `FailedEvents` (int) - Default: 0
- `LastEventReceivedAt` (DateTime, nullable)
- `Configuration` (string, 4000, nullable) - JSON string for platform-specific config
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `(Platform, SubscriptionId)` (unique)
- `TenantId`
- `SocialAccountId`
- `IsActive`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `SocialAccount` (optional)

#### 46. WebhookProcessingLogs
**Purpose**: Webhook processing history

**Columns**:
- `Id` (Guid, PK)
- `WebhookEventId` (Guid, FK → WebhookEvents)
- `ProcessingStep` (string, 100) - "Verification", "Parsing", "Forwarding"
- `Status` (string, 50) - "Success", "Failed", "Warning"
- `Message` (string, 1000, nullable)
- `ErrorDetails` (string, 4000, nullable)
- `ProcessedAt` (DateTime) - When step was processed
- `ProcessingTimeMs` (long) - Processing time in milliseconds, Default: 0
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `WebhookEventId`
- `ProcessedAt`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `WebhookEvent`

---

### Developer API Service (4 Tables)

#### 47. DeveloperApps
**Purpose**: Developer applications

**Columns**:
- `Id` (Guid, PK)
- `TenantId` (Guid)
- `UserId` (Guid) - Developer who created the app
- `Name` (string, 200)
- `Description` (string, 1000, nullable)
- `WebsiteUrl` (string, 500, nullable)
- `RedirectUri` (string, 500, nullable) - OAuth redirect URI
- `Status` (string, 50) - "Active", "Suspended", "Revoked"
- `IsActive` (bool) - Default: true
- `RateLimitPerMinute` (int) - Default: 60
- `RateLimitPerHour` (int) - Default: 1000
- `RateLimitPerDay` (int) - Default: 10000
- `TotalApiCalls` (int) - Default: 0
- `LastApiCallAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `TenantId`
- `UserId`
- `Name`
- `Status`
- `IsDeleted` (query filter)

**Relationships**:
- One-to-Many: `ApiKeys`
- One-to-Many: `ApiUsages`

#### 48. ApiKeys
**Purpose**: API keys (hashed)

**Columns**:
- `Id` (Guid, PK)
- `DeveloperAppId` (Guid, FK → DeveloperApps)
- `TenantId` (Guid)
- `KeyName` (string, 200) - User-friendly name
- `KeyHash` (string, 500) - Hashed API key (for storage)
- `KeyPrefix` (string, 20) - First 8 chars for display (e.g., "sk_live_ab")
- `IsActive` (bool) - Default: true
- `ExpiresAt` (DateTime, nullable)
- `LastUsedAt` (DateTime, nullable)
- `Scopes` (string, 1000, nullable) - JSON array of allowed scopes
- `Permissions` (string, 2000, nullable) - JSON object for granular permissions
- `RateLimitPerMinute` (int, nullable) - Override app-level limit
- `RateLimitPerHour` (int, nullable) - Override app-level limit
- `RateLimitPerDay` (int, nullable) - Override app-level limit
- `TotalCalls` (int) - Default: 0
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `DeveloperAppId`
- `TenantId`
- `KeyHash` (unique)
- `KeyPrefix`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `DeveloperApp`
- One-to-Many: `ApiUsages`
- One-to-Many: `RateLimits`

**Security**: Keys are hashed (never stored in plain text)

#### 49. ApiUsages
**Purpose**: API usage tracking

**Columns**:
- `Id` (Guid, PK)
- `DeveloperAppId` (Guid, FK → DeveloperApps)
- `ApiKeyId` (Guid, nullable, FK → ApiKeys)
- `TenantId` (Guid)
- `Endpoint` (string, 500) - e.g., "/api/posts"
- `Method` (string, 10) - "GET", "POST", "PUT", "DELETE"
- `RequestPath` (string, 1000, nullable)
- `QueryString` (string, 2000, nullable)
- `StatusCode` (int) - Default: 200
- `ResponseTimeMs` (long) - Default: 0
- `ResponseSizeBytes` (long) - Default: 0
- `UserAgent` (string, 500, nullable)
- `IpAddress` (string, 50, nullable)
- `RequestId` (string, 100, nullable) - For tracing
- `RequestedAt` (DateTime) - When request was made
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `DeveloperAppId`
- `ApiKeyId`
- `TenantId`
- `Endpoint`
- `RequestedAt`
- `RequestId`
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `DeveloperApp`
- Many-to-One: `ApiKey` (optional)

#### 50. RateLimits (DeveloperAPI)
**Purpose**: API rate limiting records (DeveloperAPI-specific)

**Columns**:
- `Id` (Guid, PK)
- `DeveloperAppId` (Guid, FK → DeveloperApps)
- `ApiKeyId` (Guid, nullable, FK → ApiKeys)
- `TenantId` (Guid)
- `WindowType` (string, 20) - "Minute", "Hour", "Day"
- `WindowStart` (DateTime) - Start of rate limit window
- `WindowEnd` (DateTime) - End of rate limit window
- `RequestCount` (int) - Current request count, Default: 0
- `Limit` (int) - Maximum requests in this window, Default: 60
- `IsExceeded` (bool) - Default: false
- `ExceededAt` (DateTime, nullable)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `DeveloperAppId`
- `ApiKeyId`
- `(DeveloperAppId, WindowType, WindowStart)` - Composite for app-level limits
- `(ApiKeyId, WindowType, WindowStart)` - Composite for key-level limits
- `IsDeleted` (query filter)

**Relationships**:
- Many-to-One: `DeveloperApp`
- Many-to-One: `ApiKey` (optional)

**Note**: Separate from `ApiRateLimits` (Shared.Kernel) which is for generic API rate limiting

---

### Shared Tables (1 Table)

#### 51. ApiRateLimits
**Purpose**: Generic API rate limits (for user, tenant, IP, endpoint-based rate limiting)

**Columns**:
- `Id` (Guid, PK)
- `Identifier` (string, 200) - UserId, TenantId, IP address, or endpoint pattern
- `IdentifierType` (string, 50) - "User", "Tenant", "IP", "Endpoint", "UserEndpoint"
- `UserId` (Guid, nullable) - For easier querying
- `TenantId` (Guid, nullable) - For easier querying
- `WindowType` (string, 20) - "Minute", "Hour", "Day"
- `WindowStart` (DateTime) - Start of rate limit window
- `WindowEnd` (DateTime) - End of rate limit window
- `RequestCount` (int) - Current request count, Default: 0
- `Limit` (int) - Maximum requests in this window, Default: 60
- `IsExceeded` (bool) - Default: false
- `ExceededAt` (DateTime, nullable)
- `EndpointPattern` (string, 500, nullable) - For endpoint-specific limits
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

**Indexes**:
- `Identifier`
- `IdentifierType`
- `UserId`
- `TenantId`
- `(Identifier, IdentifierType, WindowType, WindowStart)` - Composite for identifier-based limits
- `(UserId, WindowType, WindowStart)` - Composite for user-based limits
- `(TenantId, WindowType, WindowStart)` - Composite for tenant-based limits
- `IsDeleted` (query filter)

**Relationships**:
- None (standalone)

**Use Case**: Generic rate limiting for API endpoints, user-based limits, tenant-based limits, IP-based limits

---

## Relationships

### Key Entity Relationships

#### 1. Tenant Hierarchy
```
Tenant (1) ──→ (Many) Users
Tenant (1) ──→ (Many) Subscriptions (1:1 unique)
Tenant (1) ──→ (Many) Clients
Tenant (1) ──→ (Many) Workspaces
Tenant (1) ──→ (Many) SocialAccounts
```

#### 2. User Relationships
```
User (1) ──→ (1) UserSettings (1:1 unique)
User (1) ──→ (Many) RefreshTokens
User (1) ──→ (Many) Posts (CreatedByUserId)
User (1) ──→ (Many) CreatedUsers (self-reference, CreatedByUserId)
User (1) ──→ (Many) UpdatedUsers (self-reference, UpdatedByUserId)
User (Many) ──→ (1) Client (ClientId, for client users)
```

#### 3. Post Relationships
```
Post (1) ──→ (Many) PostTargets
Post (1) ──→ (Many) PostVersions
Post (1) ──→ (Many) ApprovalRequests
Post (1) ──→ (Many) ContentScheduledJobs
Post (1) ──→ (Many) ContentPublishLogs
Post (1) ──→ (Many) ContentAnalyticsRecords
Post (1) ──→ (Many) SchedulerScheduledJobs
Post (1) ──→ (Many) SchedulerPublishLogs
Post (1) ──→ (Many) AnalyticsServiceRecords
Post (1) ──→ (1) PostPerformances (1:1 unique)
Post (Many) ──→ (1) Client
Post (Many) ──→ (1) MediaAsset (optional)
```

#### 4. Social Account Relationships
```
SocialAccount (1) ──→ (Many) SocialAccountPages
SocialAccount (1) ──→ (Many) SocialAccountTokens
SocialAccount (1) ──→ (Many) PostTargets
SocialAccount (1) ──→ (Many) WebhookSubscriptions (optional)
```

#### 5. Workspace Relationships
```
Workspace (1) ──→ (Many) TeamMembers
Workspace (1) ──→ (Many) Comments
Workspace (1) ──→ (Many) Tasks
Workspace (1) ──→ (Many) CollaborationRequests
Workspace (1) ──→ (Many) ActivityLogs
```

#### 6. Subscription Relationships
```
Subscription (1) ──→ (Many) Invoices
Subscription (1) ──→ (Many) TrialUsages
Subscription (1) ──→ (1) Tenant (1:1 unique)
```

#### 7. Analytics Relationships
```
AnalyticsServiceRecord (1) ──→ (Many) EngagementSnapshots
AnalyticsServiceRecord (Many) ──→ (1) Post
```

#### 8. Developer API Relationships
```
DeveloperApp (1) ──→ (Many) ApiKeys
DeveloperApp (1) ──→ (Many) ApiUsages
DeveloperApp (1) ──→ (Many) RateLimits (DeveloperAPI)
```

#### 9. Webhook Relationships
```
WebhookEvent (1) ──→ (Many) WebhookProcessingLogs
```

#### 10. Scheduler Relationships
```
ScheduledJob (1) ──→ (Many) PublishLogs
ScheduledJob (Many) ──→ (1) Post (optional)
```

### Foreign Key Constraints

All foreign keys use appropriate delete behaviors:
- **Cascade**: Child records deleted when parent is deleted (e.g., PostTargets → Post)
- **Restrict**: Prevents deletion if child records exist (e.g., Users → Tenant)
- **SetNull**: Sets foreign key to null when parent is deleted (e.g., Post → MediaAsset)
- **NoAction**: No automatic action (e.g., ApiUsage → DeveloperApp)

---

## Indexes

### Index Strategy

All tables use Entity Framework Core's `HasIndex()` method to create indexes. Indexes are created for:

1. **Primary Keys**: All tables have `Id` as primary key (Guid)
2. **Foreign Keys**: All foreign key columns are indexed
3. **Unique Constraints**: Email, PlanId, InvoiceNumber, etc.
4. **Query Filters**: `IsDeleted` column for soft delete filtering
5. **Composite Indexes**: For common query patterns (e.g., `(PostId, Platform)`, `(Status, ScheduledAt)`)

### Common Index Patterns

#### Soft Delete Filter
All tables have a query filter on `IsDeleted`:
```csharp
entity.HasQueryFilter(e => !e.IsDeleted);
```

#### Composite Indexes for Queries
- `(PostId, Platform)` - Unique constraint for AnalyticsServiceRecords
- `(Status, ScheduledAt)` - For job processing queries
- `(UserId, IsRead)` - For notification queries
- `(WorkspaceId, UserId)` - Unique constraint for TeamMembers
- `(Platform, SubscriptionId)` - Unique constraint for WebhookSubscriptions

#### Date/Time Indexes
- `ScheduledAt` - For scheduled job queries
- `CreatedAt` - For chronological queries
- `PublishedAt` - For analytics queries
- `ReceivedAt` - For webhook queries

---

## Migrations

### Migration Location
**Path**: `NexusPost/src/Shared/Shared.Infrastructure/Migrations/`

### Key Migrations

1. **`20251116071155_ConsolidatedDbContext`**
   - Initial consolidated database migration
   - Creates all base tables for all services

2. **`20251116132233_UpdateTaskAssignmentForAgencyTasks`**
   - Adds `ClientId` to Tasks table
   - Enables agency task assignment to clients

3. **`20251116140533_AddClientIdToUser`**
   - Adds `ClientId` to Users table
   - Enables client user linking

4. **`20251205080742_AddApiRateLimitsTable`**
   - Creates `ApiRateLimits` table
   - Adds generic rate limiting support

5. **`20251205121537_AddEmailQueueAndTenantBranding`**
   - Creates `EmailQueues` table
   - Adds tenant branding fields (LogoUrl, PrimaryColor, SecondaryColor)

6. **`20251206012834_AddDeviceTokensTable`**
   - Creates `DeviceTokens` table
   - Adds push notification support

7. **`20251208054156_AddPublicIdAndThumbnailUrlToMediaAsset`**
   - Adds `PublicId` and `ThumbnailUrl` to MediaAssets
   - Improves media asset management

### How Migrations Work

1. **Automatic Migration on Startup**
   - Migrations run automatically in `Program.cs` → `RunMigrationsAsync()`
   - Retries up to 3 times with exponential backoff
   - Handles Azure SQL connection issues gracefully

2. **Creating New Migrations**
   ```bash
   cd NexusPost/src/Shared/Shared.Infrastructure
   dotnet ef migrations add MigrationName --context NexusPostDbContext
   ```

3. **Applying Migrations**
   - Migrations are applied automatically on application startup
   - Can also be applied manually:
   ```bash
   dotnet ef database update --context NexusPostDbContext
   ```

4. **Migration History**
   - Stored in `__EFMigrationsHistory` table
   - Tracks which migrations have been applied

---

## Data Modifications

### Soft Delete Pattern

**All tables implement soft delete** using the `IsDeleted` boolean column:

- **Query Filter**: Entity Framework Core automatically filters out deleted records
- **Implementation**: `entity.HasQueryFilter(e => !e.IsDeleted);`
- **Benefits**:
  - Data is preserved for audit purposes
  - Can be restored if needed
  - Maintains referential integrity

### Audit Fields

All tables inherit from `BaseEntity` which includes:
- `Id` (Guid, PK)
- `IsDeleted` (bool)
- `CreatedAt` (DateTime)
- `UpdatedAt` (DateTime, nullable)

### Data Seeding

#### On Application Startup

1. **Admin User**
   - Creates default admin user if not exists
   - Email: `admin@nexuspost.com`
   - Password: Hashed with BCrypt

2. **Billing Plans**
   - Seeds 4 billing plans:
     - **Basic**: $29.99/month
     - **Pro**: $79.99/month
     - **Enterprise**: $199.99/month
     - **Agency**: $49.99/month per account

3. **System Tenant**
   - Creates system tenant for admin operations
   - TenantType: "System"

### Data Modification Patterns

#### 1. Create Operations
- All creates set `CreatedAt = DateTime.UtcNow`
- `IsDeleted = false` by default
- `Id` is auto-generated (Guid)

#### 2. Update Operations
- Updates set `UpdatedAt = DateTime.UtcNow`
- Soft delete sets `IsDeleted = true` (does not delete record)

#### 3. Delete Operations
- **Soft Delete**: Sets `IsDeleted = true`
- **Hard Delete**: Not recommended (use soft delete)
- Cascade deletes handled by Entity Framework Core

#### 4. Bulk Operations
- Use `ExecuteInTransactionAsync()` for bulk operations
- Supports retry strategy for Azure SQL

### Data Consistency

#### Transaction Management
- Unit of Work pattern implemented via `IUnitOfWork`
- Transactions managed in `NexusPostDbContext`
- Supports retry strategies for Azure SQL

#### Referential Integrity
- Foreign keys enforce referential integrity
- Delete behaviors configured per relationship:
  - **Cascade**: Child records deleted with parent
  - **Restrict**: Prevents deletion if children exist
  - **SetNull**: Sets FK to null when parent deleted
  - **NoAction**: No automatic action

### Data Retention

#### Soft Deleted Records
- Soft deleted records are retained indefinitely
- Can be purged via background job (future enhancement)
- Query filter automatically excludes deleted records

#### Log Tables
- `PublishLogs`, `WebhookProcessingLogs`, `ApiUsages` are append-only
- Can be archived after retention period (future enhancement)

---

## Summary

### Total Tables: 51

**By Service**:
- Auth Service: 4 tables
- Social Account Service: 4 tables
- Content Service: 9 tables
- Billing Service: 5 tables
- Collab Service: 6 tables
- AI Service: 4 tables
- Analytics Service: 3 tables
- Notification Service: 5 tables
- Scheduler Service: 3 tables
- Webhook Service: 3 tables
- Developer API Service: 4 tables
- Shared Tables: 1 table

### Key Features

1. **Single Shared Database**: All services share `NexusPostDbContext`
2. **Soft Delete**: All tables implement soft delete pattern
3. **Audit Fields**: All tables have `CreatedAt` and `UpdatedAt`
4. **Indexes**: Comprehensive indexing for performance
5. **Relationships**: Well-defined foreign key relationships
6. **Migrations**: Automatic migration on startup
7. **Data Seeding**: Admin user and billing plans seeded on startup

### Database Modifications

- **Migrations**: Managed via Entity Framework Core
- **Automatic**: Migrations run on application startup
- **Retry Logic**: Handles Azure SQL connection issues
- **Version Control**: Migration history tracked in `__EFMigrationsHistory`

---

**Last Updated**: December 2024
**Version**: 1.0.0
