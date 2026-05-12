# API 鉴权体系设计报告

## 概述

Formbricks API 支持三种鉴权身份，通过统一的中间件架构进行认证、范围解析和限流控制。

| 身份类型 | 认证方式 | 适用场景 |
|---------|---------|---------|
| API Key | `x-api-key` 请求头 | 组织共用的集成、服务端调用 |
| Session | Cookie (NextAuth) | 后台界面用户操作 |
| Personal Token | (待实现) | 开发者个人调用 |

---

## 一、鉴权中间件架构

### 1.1 统一 API Wrapper 模式

所有 API 端点通过高阶函数包装，实现认证、验证、限流、审计的横切关注点。

**V3 API 架构 (`apps/web/app/api/v3/lib/api-wrapper.ts`)**

```typescript
type TV3AuthMode = "none" | "session" | "apiKey" | "both";

export const withV3ApiWrapper = ({
  auth = "both",           // 认证模式
  schemas,                 // Zod 验证 schema
  rateLimit = true,        // 是否启用限流
  customRateLimitConfig,   // 自定义限流配置
  action, targetType,      // 审计日志配置
  handler,                 // 业务处理函数
}) => {
  // 1. 认证
  // 2. 输入验证
  // 3. 限流
  // 4. 审计日志
  // 5. 执行业务逻辑
};
```

**V2 API 架构 (`apps/web/modules/api/v2/auth/api-wrapper.ts`)**

```typescript
export const apiWrapper = async ({
  request,
  schemas,
  rateLimit = true,
  handler,
}) => {
  // V2 仅支持 API Key 认证
  const authentication = await authenticateRequest(request);
  // ...
};
```

### 1.2 认证流程

**V3 认证优先级 (`authenticateV3Request`)**

```
请求到达
    │
    ▼
  ┌─────────────────────────────────┐
  │ 检查 auth mode                  │
  ├─────────────────────────────────┤
  │ "both" 且有 x-api-key 头?       │
  │   ├─ 是 → 尝试 API Key 认证     │
  │   └─ 否 → 尝试 Session 认证     │
  │ "session"? → Session 认证       │
  │ "apiKey"?  → API Key 认证       │
  │ "none"?    → 跳过认证           │
  └─────────────────────────────────┘
```

**代码实现要点**

```typescript
// V3 支持混合认证
async function authenticateV3Request(req, authMode) {
  // 优先 API Key
  if (authMode === "both" && req.headers.has("x-api-key")) {
    const apiKeyAuth = await authenticateRequest(req);
    if (apiKeyAuth) return apiKeyAuth;
  }

  // 其次 Session
  if (authMode === "session" || authMode === "both") {
    const session = await getServerSession(authOptions);
    if (session?.user?.id) return session;
  }

  // 兜底 API Key
  if (authMode === "apiKey" || authMode === "both") {
    return await authenticateRequest(req);
  }
}
```

**V2 API Key 认证 (`authenticate-request.ts`)**

```typescript
export const authenticateRequest = async (request) => {
  const apiKey = request.headers.get("x-api-key");
  if (!apiKey) return err({ type: "unauthorized" });

  const apiKeyData = await getApiKeyWithPermissions(apiKey);
  if (!apiKeyData) return err({ type: "unauthorized" });

  return ok({
    type: "apiKey",
    environmentPermissions: [...], // 各环境的权限列表
    apiKeyId: apiKeyData.id,
    organizationId: apiKeyData.organizationId,
    organizationAccess: apiKeyData.organizationAccess,
  });
};
```

---

## 二、可用范围解析

### 2.1 权限模型

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

**权限校验函数 (`auth.ts`)**

```typescript
function apiKeyPermissionAllows(permission, minPermission): boolean {
  const grantedRank = { read: 1, write: 2, manage: 3 }[permission];
  const requiredRank = { read: 1, readWrite: 2, manage: 3 }[minPermission];
  return grantedRank >= requiredRank;
}
```

### 2.2 工作区访问控制 (V3)

`requireV3WorkspaceAccess` 统一处理两种身份的权限校验：

```typescript
export async function requireV3WorkspaceAccess(
  authentication,
  workspaceId,
  minPermission,
  requestId,
  instance
) {
  // Session 用户: 检查组织角色 + 项目团队权限
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

  // API Key: 检查环境级别的权限
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

### 2.3 认证对象类型定义

```typescript
// packages/types/auth.ts
type TAuthenticationApiKey = {
  type: "apiKey";
  environmentPermissions: Array<{
    environmentId: string;
    environmentType: "development" | "production";
    projectId: string;
    projectName: string;
    permission: "read" | "write" | "manage";
  }>;
  apiKeyId: string;
  organizationId: string;
  organizationAccess: TOrganizationAccess;
};

type TAuthSession = {
  user: TUser;
  // Session 包含完整用户信息
};
```

---

## 三、限流策略

### 3.1 核心限流实现

**原子化 Redis + Lua 脚本限流 (`rate-limit.ts`)**

```typescript
// 使用 Lua 脚本保证 INCR + EXPIRE 的原子性
// 防止多实例部署下的竞态条件
const luaScript = `
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local ttl = tonumber(ARGV[2])
  
  local current = redis.call('INCR', key)
  
  -- 仅在第一次请求时设置过期时间
  -- 避免窗口被不断延长
  if current == 1 then
    redis.call('EXPIRE', key, ttl)
  end
  
  return {current, current <= limit and 1 or 0}
`;

export const checkRateLimit = async (config, identifier) => {
  const key = createCacheKey.rateLimit.core(
    config.namespace,
    identifier,
    windowStart
  );
  
  const result = await redis.eval(luaScript, {
    keys: [key],
    arguments: [config.allowedPerInterval, ttlSeconds],
  });
  
  // ... 返回限流结果
};
```

### 3.2 按身份类型的限流标识符

```typescript
// V3 API wrapper: getRateLimitIdentifier
function getRateLimitIdentifier(authentication): string | null {
  if (!authentication) return null;

  // Session 用户: 按 userId 限流
  if ("user" in authentication && authentication.user?.id) {
    return authentication.user.id;
  }

  // API Key: 按 apiKeyId 限流
  if ("apiKeyId" in authentication) {
    return authentication.apiKeyId;
  }

  // 未认证: 按 IP hash 限流
  return null;
}
```

**IP 限流（未认证请求）**

```typescript
export const applyIPRateLimit = async (config) => {
  const ip = await getClientIpFromHeaders();
  const identifier = await hashString(ip); // 哈希保护隐私
  return await applyRateLimit(config, identifier);
};
```

### 3.3 限流配置矩阵

```typescript
// rate-limit-configs.ts
export const rateLimitConfigs = {
  // 认证端点: 严格限制防暴力破解
  auth: {
    login:          { interval: 900,  allowedPerInterval: 10, namespace: "auth:login" },      // 15分钟10次
    signup:         { interval: 3600, allowedPerInterval: 30, namespace: "auth:signup" },     // 1小时30次
    forgotPassword: { interval: 3600, allowedPerInterval: 5,  namespace: "auth:forgot" },     // 1小时5次
  },

  // API 端点: 合理业务配额
  api: {
    v1: { interval: 60, allowedPerInterval: 100, namespace: "api:v1" },     // 每分钟100次
    v2: { interval: 60, allowedPerInterval: 100, namespace: "api:v2" },
    v3: { interval: 60, allowedPerInterval: 100, namespace: "api:v3" },
  },

  // 服务端操作: 保护敏感资源
  actions: {
    emailUpdate:       { interval: 3600, allowedPerInterval: 3,  namespace: "action:email" },
    accountDeletion:   { interval: 3600, allowedPerInterval: 5,  namespace: "action:account-delete" },
    licenseRecheck:    { interval: 60,   allowedPerInterval: 5,  namespace: "action:license-recheck" },
  },
};
```

### 3.4 限流调用链

```
V3 API 请求
    │
    ▼
withV3ApiWrapper
    │
    ├─ authenticateV3Request
    │   └─ 获得 authentication 对象
    │
    ├─ parseV3Input (参数验证)
    │
    ├─ applyV3RateLimitOrRespond
    │   ├─ getRateLimitIdentifier
    │   │   └─ 根据身份类型选择标识符
    │   └─ applyRateLimit(config, identifier)
    │       └─ checkRateLimit (Redis + Lua)
    │
    └─ handler (业务逻辑)
```

---

## 四、三种鉴权身份的限流策略对比

| 身份类型 | 限流标识符 | 限流粒度 | 典型配置 |
|---------|-----------|---------|---------|
| **API Key** | `apiKeyId` | 按组织/项目 | 100 req/min (v2/v3) |
| **Session** | `userId` | 按用户 | 100 req/min |
| **Personal Token** | `tokenId` / `userId` | 按开发者 | (同 API Key) |
| **未认证** | IP 哈希 | 按客户端 IP | 更严格的认证端点限制 |

**设计原则**：
1. **API Key**: 按 key 限流，支持多服务共享同一配额
2. **Session**: 按用户限流，防止单用户滥用
3. **Personal Token**: 可按 token 或用户聚合，便于开发者管理
4. **失败开放**: Redis 不可用时跳过限流，保证可用性

---

## 五、审计日志集成

限流和认证事件都集成到审计系统：

```typescript
function buildV3AuditLog(authentication, action, targetType, apiUrl) {
  const auditLog = buildAuditLogBaseObject(action, targetType, apiUrl);

  // Session 用户
  if ("user" in authentication && authentication.user?.id) {
    auditLog.userId = authentication.user.id;
    auditLog.userType = "user";
  }
  // API Key
  else if ("apiKeyId" in authentication) {
    auditLog.userId = authentication.apiKeyId;
    auditLog.userType = "api";
    auditLog.organizationId = authentication.organizationId;
  }

  return auditLog;
}
```

---

## 六、扩展考虑: Personal Token

**添加 Personal Token 支持的最小改动**：

1. **认证层 (`authenticateV3Request`)**
   - 检测 `Authorization: Bearer <token>` 头
   - 查询 personal_token 表验证 token
   - 返回 `type: "personalToken"` 的认证对象

2. **范围解析层**
   - Token 关联到 userId，复用 Session 的权限检查逻辑
   - 支持 token 级别的额外权限限制

3. **限流层**
   - 标识符优先级: tokenId > userId
   - 可配置是否与用户共享配额

```typescript
// 扩展后的 getRateLimitIdentifier
function getRateLimitIdentifier(authentication) {
  if ("personalTokenId" in authentication) {
    return authentication.personalTokenId; // 独立配额
    // return authentication.userId; // 或与用户共享配额
  }
  // ... 其他逻辑
}
```
