# API 鉴权体系设计文档

> **文档状态**：基于代码仓库真实实现编写，标注已实现 vs 待实现

---

## 概述

| 身份类型 | 认证方式 | 实现状态 | 限流标识符 |
|---------|---------|---------|-----------|
| **API Key** | `x-api-key` 请求头 | ✅ 已实现 | `apiKeyId` |
| **Session** | NextAuth Cookie | ✅ 已实现 | `userId` |
| **Personal Token** | `Authorization: Bearer <token>` | ❌ 未实现 | 设计方案见下文 |

---

## 一、鉴权中间件架构

### 1.1 统一 API Wrapper 模式

当前存在三套独立的 API 认证包装器，**不共享主链路**：

| Wrapper | 适用范围 | 认证支持 | 实现文件 |
|---------|---------|---------|---------|
| `withV3ApiWrapper` | V3 管理 API | Session + API Key | `apps/web/app/api/v3/lib/api-wrapper.ts` |
| `withV1ApiWrapper` | V1/V2 Client + Management API | Session / API Key / None | `apps/web/app/lib/api/with-api-logging.ts` |
| `apiWrapper` (V2) | V2 Management API | API Key only | `apps/web/modules/api/v2/auth/api-wrapper.ts` |

**事实依据**：
- V3 使用独立的 `authenticateV3Request` 函数（line 126-154）
- V1 使用 `handleAuthentication` 函数根据路由类型决定认证方式（line 143-159）
- V2 只调用 `authenticateRequest` 验证 API Key（line 58）

### 1.2 认证流程按入口分列

#### 🔹 入口一：V3 API 认证流程

```
请求 → withV3ApiWrapper
    ↓
    [鉴权] authenticateV3Request (line 126-154)
    ├─ 检测 authMode: "none" | "session" | "apiKey" | "both"
    ├─ "both" 且有 x-api-key 头 → 尝试 API Key 认证
    ├─ 否则尝试 Session 认证 (getServerSession)
    └─ 兜底再尝试 API Key 认证
    ↓
    [输入验证] parseV3Input
    ↓
    [限流] applyV3RateLimitOrRespond
    ↓
    业务 handler
```

**事实依据**：`apps/web/app/api/v3/lib/api-wrapper.ts:358-422`

#### 🔹 入口二：V1/V2 API 认证流程

```
请求 → withV1ApiWrapper
    ↓
    [路由分类] getRouteType
    ├─ Client API → None 认证 + IP 限流
    ├─ Management API → 根据 endpoint-validator 决定: ApiKey / Session / Both
    └─ Integration API → Session 认证
    ↓
    [鉴权] handleAuthentication
    ├─ ApiKey → authenticateRequest
    ├─ Session → getServerSession
    ├─ Both → session ?? authenticateRequest
    └─ None → return null
    ↓
    [限流] handleRateLimiting
    ↓
    业务 handler
```

**事实依据**：`apps/web/app/lib/api/with-api-logging.ts:230-302`

#### 🔹 入口三：Server Actions 认证流程

```
调用 → actionClient / authenticatedActionClient
    ↓
    [鉴权] authenticatedActionClient middleware (line 51-65)
    └─ getServerSession → 验证 user
    ↓
    业务逻辑（各 action 自行调用限流）
```

**事实依据**：`apps/web/lib/utils/action-client/index.ts:51-65`

---

## 二、可用范围解析

### 2.1 权限模型（已实现）

**API Key 权限等级**

| 权限 | 等级 | 说明 |
|-----|-----|-----|
| `read` | 1 | 只读访问 |
| `write` | 2 | 读写访问 |
| `manage` | 3 | 管理权限 |

**Session 用户权限**

| 权限 | 等级 |
|-----|-----|
| `read` | 1 |
| `readWrite` | 2 |
| `manage` | 3 |

**权限校验函数**：
```typescript
// apps/web/app/api/v3/lib/auth.ts:14-28
function apiKeyPermissionAllows(permission, minPermission): boolean {
  const grantedRank = { read: 1, write: 2, manage: 3 }[permission];
  const requiredRank = { read: 1, readWrite: 2, manage: 3 }[minPermission];
  return grantedRank >= requiredRank;
}
```

### 2.2 工作区访问控制（V3 已实现）

`requireV3WorkspaceAccess` 统一处理两种身份的权限校验：

```typescript
export async function requireV3WorkspaceAccess(
  authentication, workspaceId, minPermission, requestId, instance
) {
  // Session 用户
  if ("user" in authentication && authentication.user?.id) {
    const context = await resolveV3WorkspaceContext(workspaceId);
    await checkAuthorizationUpdated({
      userId: authentication.user.id,
      organizationId: context.organizationId,
      access: [
        { type: "organization", roles: ["owner", "manager"] },
        { type: "projectTeam", projectId: context.projectId, minPermission },
      ],
    });
    return context;
  }

  // API Key
  const keyAuth = authentication as TAuthenticationApiKey;
  const context = await resolveV3WorkspaceContext(workspaceId);
  const permission = keyAuth.environmentPermissions.find(
    (env) => env.environmentId === context.environmentId
  );

  if (!permission || !apiKeyPermissionAllows(permission.permission, minPermission)) {
    return problemForbidden(...);
  }

  return context;
}
```

**事实依据**：`apps/web/app/api/v3/lib/auth.ts:80-121`

### 2.3 Personal Token：未实现但可行的设计方案

**现状缺口**：
- 代码库中**不存在** PersonalToken 数据表模型
- **不存在** `Authorization: Bearer <token>` 解析逻辑
- **不存在** token 过期、撤销、权限范围等机制

**实现方案设计**：

#### 1. 数据库模型扩展
```prisma
// prisma.schema 新增模型（当前不存在）
model PersonalToken {
  id           String   @id @default(cuid())
  userId       String
  name         String   // Token 名称，如 "GitHub Actions"
  tokenHash    String   @unique // SHA-256 哈希存储
  permissions  Json     // 可选的精细权限控制
  createdAt    DateTime @default(now())
  lastUsedAt   DateTime?
  expiresAt    DateTime? // 可选过期时间
  isRevoked    Boolean  @default(false)

  user         User     @relation(fields: [userId], references: [id])
}
```

#### 2. 认证层扩展
修改 V3 的 `authenticateV3Request` 函数，增加 Bearer token 检测：

```typescript
async function authenticateV3Request(req, authMode) {
  // 现有逻辑...

  // + 新增 Personal Token 检测
  const authHeader = req.headers.get("authorization");
  if (authHeader?.startsWith("Bearer ")) {
    const token = authHeader.slice(7);
    const personalToken = await validatePersonalToken(token);
    if (personalToken) {
      return {
        type: "personalToken",
        personalTokenId: personalToken.id,
        userId: personalToken.userId,
        user: { id: personalToken.userId }, // 兼容 Session 权限检查
      };
    }
  }

  // 现有逻辑...
}
```

#### 3. 权限检查兼容性
Personal Token 关联到 `userId`，可直接复用现有 Session 权限检查逻辑，无需修改 `requireV3WorkspaceAccess`。

---

## 三、限流策略

### 3.1 核心限流实现（已实现）

**原子化 Redis + Lua 脚本限流**

```typescript
// apps/web/modules/core/rate-limit/rate-limit.ts
const luaScript = `
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local ttl = tonumber(ARGV[2])

  local current = redis.call('INCR', key)

  -- 仅在第一次请求时设置 TTL，避免窗口延长
  if current == 1 then
    redis.call('EXPIRE', key, ttl)
  end

  return {current, current <= limit and 1 or 0}
`;
```

**事实依据**：`apps/web/modules/core/rate-limit/rate-limit.ts:46-61`

### 3.2 按身份类型的限流入口（分列展示）

#### 🔹 限流入口一：未认证请求 → IP 哈希限流

| 场景 | 调用位置 | 限流配置 |
|------|---------|---------|
| Client API (SDK 请求) | `withV1ApiWrapper` → `applyClientRateLimit` | `rateLimitConfigs.api.client` |
| 用户注册 action | `signup/actions.ts:234` | `rateLimitConfigs.auth.signup` |
| 忘记密码 action | `forgot-password/actions.ts:21` | `rateLimitConfigs.auth.forgotPassword` |
| 发送问卷链接邮件 | `survey/link/actions.ts:17` | `rateLimitConfigs.actions.sendLinkSurveyEmail` |
| 登录 (next-auth) | 需单独配置，暂无统一入口 | - |

**实现函数**：
```typescript
// apps/web/modules/core/rate-limit/helpers.ts
export const applyIPRateLimit = async (config) => {
  const ip = await getClientIpFromHeaders();
  const identifier = await hashString(ip); // 哈希保护隐私
  return await applyRateLimit(config, identifier);
};
```

#### 🔹 限流入口二：Session 身份 → userId 限流

| 场景 | 调用位置 | 限流配置 |
|------|---------|---------|
| V1 Integration API | `withV1ApiWrapper` → `handleRateLimiting` line 74-76 | `rateLimitConfigs.api.v1` |
| V1 Management API (Session 模式) | `withV1ApiWrapper` → line 74-76 | `rateLimitConfigs.api.v1` |
| V3 API (Session 模式) | `withV3ApiWrapper` → `getRateLimitIdentifier` line 90-104 | `rateLimitConfigs.api.v3` |
| Storage DELETE (Session) | `storage/.../route.ts:109` | `rateLimitConfigs.storage.delete` |
| V1 Me endpoint | `management/me/route.ts:175` | `rateLimitConfigs.api.v1` |
| 账户删除（密码方式） | `DeleteAccountModal/actions.ts:56` | `rateLimitConfigs.actions.accountDeletion` |
| 账户删除（SSO 重认证） | `DeleteAccountModal/actions.ts:36` | `rateLimitConfigs.actions.accountDeletion` |
| 邮箱更新 | `profile/actions.ts:41` | `rateLimitConfigs.actions.emailUpdate` |
| License 重新检查 | `license-check/actions.ts:45` | `rateLimitConfigs.actions.licenseRecheck` |

**标识符解析**：
```typescript
// V3 - apps/web/app/api/v3/lib/api-wrapper.ts:95-97
if ("user" in authentication && authentication.user?.id) {
  return authentication.user.id;
}

// V1 - apps/web/app/lib/api/with-api-logging.ts:74-76
if ("user" in authentication) {
  await applyRateLimit(config, authentication.user.id);
}
```

#### 🔹 限流入口三：API Key 身份 → apiKeyId 限流

| 场景 | 调用位置 | 限流配置 |
|------|---------|---------|
| V1 Management API (API Key) | `withV1ApiWrapper` → `handleRateLimiting` line 77-79 | `rateLimitConfigs.api.v1` |
| V2 Management API | `api-wrapper.ts:125` | `rateLimitConfigs.api.v2` |
| V3 API (API Key) | `withV3ApiWrapper` → `getRateLimitIdentifier` line 99-101 | `rateLimitConfigs.api.v3` |
| Storage DELETE (API Key) | `storage/.../route.ts:107` | `rateLimitConfigs.storage.delete` |
| V1 Me endpoint | `management/me/route.ts:158` | `rateLimitConfigs.api.v1` |

**标识符解析**：
```typescript
// V3 - apps/web/app/api/v3/lib/api-wrapper.ts:99-101
if ("apiKeyId" in authentication) {
  return authentication.apiKeyId;
}

// V1 - apps/web/app/lib/api/with-api-logging.ts:77-79
if ("apiKeyId" in authentication) {
  await applyRateLimit(config, authentication.apiKeyId);
}
```

#### 🔹 限流入口四：Server Actions（组织级别限流）

| 场景 | 调用位置 | 限流配置 | 标识符 |
|------|---------|---------|-------|
| 问卷跟进邮件发送 | `follow-ups/lib/follow-ups.ts:196` | `rateLimitConfigs.actions.surveyFollowUp` | `organization.id` |

### 3.3 限流配置矩阵（已实现，真实值）

```typescript
// apps/web/modules/core/rate-limit/rate-limit-configs.ts
export const rateLimitConfigs = {
  auth: {
    login:          { interval: 900,  allowedPerInterval: 10, namespace: "auth:login" },      // 15分钟10次
    signup:         { interval: 3600, allowedPerInterval: 30, namespace: "auth:signup" },     // 1小时30次
    forgotPassword: { interval: 3600, allowedPerInterval: 5,  namespace: "auth:forgot" },     // 1小时5次
    verifyEmail:    { interval: 3600, allowedPerInterval: 10, namespace: "auth:verify" },
  },
  api: {
    v1:     { interval: 60, allowedPerInterval: 100, namespace: "api:v1" },     // 每分钟100次
    v2:     { interval: 60, allowedPerInterval: 100, namespace: "api:v2" },
    v3:     { interval: 60, allowedPerInterval: 100, namespace: "api:v3" },
    client: { interval: 60, allowedPerInterval: 100, namespace: "api:client" },
  },
  actions: {
    emailUpdate:          { interval: 3600, allowedPerInterval: 3,  namespace: "action:email" },             // 每小时3次
    accountDeletion:      { interval: 3600, allowedPerInterval: 5,  namespace: "action:account-delete" },    // 每小时5次
    surveyFollowUp:       { interval: 3600, allowedPerInterval: 50, namespace: "action:followup" },          // 每小时50次
    sendLinkSurveyEmail:  { interval: 3600, allowedPerInterval: 10, namespace: "action:send-link-survey-email" }, // 每小时10次
    licenseRecheck:       { interval: 60,   allowedPerInterval: 5,  namespace: "action:license-recheck" },   // 每分钟5次
  },
  storage: {
    upload: { interval: 60, allowedPerInterval: 5, namespace: "storage:upload" }, // 每分钟5次
    delete: { interval: 60, allowedPerInterval: 5, namespace: "storage:delete" }, // 每分钟5次
  },
} as const;
```

### 3.4 配置键 - Namespace - 调用位置对照表

| 配置键 | Namespace（真实值） | 调用位置（精确行号） |
|-------|---------------------|---------------------|
| `auth.login` | `auth:login` | `modules/auth/lib/authOptions.ts:191` |
| `auth.signup` | `auth:signup` | `modules/auth/signup/actions.ts:234` |
| `auth.forgotPassword` | `auth:forgot` | `modules/auth/forgot-password/actions.ts:21` |
| `auth.verifyEmail` | `auth:verify` | `modules/auth/lib/authOptions.ts:381`, `modules/auth/verification-requested/actions.ts:55` |
| `api.v1` | `api:v1` | `app/lib/api/with-api-logging.ts:74-79`, `app/api/v1/management/me/route.ts:158,175` |
| `api.v2` | `api:v2` | `modules/api/v2/auth/api-wrapper.ts:125` |
| `api.v3` | `api:v3` | `app/api/v3/lib/api-wrapper.ts:90-104` |
| `api.client` | `api:client` | `app/lib/api/with-api-logging.ts:61` |
| `actions.emailUpdate` | `action:email` | `app/(app)/environments/[environmentId]/settings/(account)/profile/actions.ts:41` |
| `actions.accountDeletion` | `action:account-delete` | `modules/account/components/DeleteAccountModal/actions.ts:36,56` |
| `actions.surveyFollowUp` | `action:followup` | `modules/survey/follow-ups/lib/follow-ups.ts:196` |
| `actions.sendLinkSurveyEmail` | `action:send-link-survey-email` | `modules/survey/link/actions.ts:17` |
| `actions.licenseRecheck` | `action:license-recheck` | `modules/ee/license-check/actions.ts:45` |
| `storage.upload` | `storage:upload` | `app/api/v1/management/storage/route.ts:77`, `app/api/v1/client/[environmentId]/storage/route.ts:112` |
| `storage.delete` | `storage:delete` | `app/storage/[environmentId]/[accessType]/[fileName]/route.ts:107,109` |

### 3.5 Personal Token 限流：设计方案

**方案 A：Token 独立配额（推荐）**
```typescript
// 扩展 getRateLimitIdentifier
function getRateLimitIdentifier(authentication) {
  // + 新增 Personal Token 分支
  if ("personalTokenId" in authentication) {
    return authentication.personalTokenId; // 每个 token 独立配额
  }
  // 现有逻辑...
}
```

**方案 B：与用户共享配额**
```typescript
if ("personalTokenId" in authentication) {
  return authentication.userId; // 与该用户的 Session 共享同一配额
}
```

---

## 四、待改进与统一化建议

### 当前架构问题

| 问题 | 说明 |
|-----|-----|
| 三套 Wrapper 并存 | V1/V2/V3 各有一套认证/限流逻辑，维护成本高 |
| 限流入口分散 | 未形成统一限流中间件，各模块自行调用易遗漏 |
| Personal Token 缺失 | 开发者无法通过 token 方式访问，只能使用组织级 API Key |

### 统一化建议

1. **统一认证中间件**：重构 `withV3ApiWrapper` 作为唯一标准，支持三种身份类型
2. **统一限流层**：在认证后统一调用限流，无需各 handler 自行处理
3. **实现 Personal Token**：基于现有 Session 权限体系快速落地
