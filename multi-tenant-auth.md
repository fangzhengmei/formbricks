# Formbricks 多租户权限体系报告

## 一、整体权限层级架构

Formbricks 采用四层权限架构，从上到下依次为：

```
User (用户)
   ↓
Organization (组织)
   ↓
Team (团队) → Project (项目)
                     ↓
                Environment (环境)
```

**核心判定逻辑**：

1. **组织级访问**：通过 `Membership` 表建立用户与组织的关联，带有 `role` 字段
2. **团队级访问**：通过 `TeamUser` 表建立用户与团队的关联
3. **项目级访问**：通过 `ProjectTeam` 表建立团队与项目的关联
4. **环境级访问**：环境从属于项目，访问权限继承自上层

---

## 二、会话解析 (Session Resolution)

### 2.1 认证流程

**技术栈**：NextAuth.js (`next-auth`) + 数据库会话策略

#### 登录认证流程

```
用户提交凭证
    ↓
applyIPRateLimit → 防暴力破解
    ↓
Prisma 查找用户 (email 唯一)
    ↓
bcrypt 密码验证 (12轮哈希, constant-time 验证防止时序攻击)
    ↓
2FA 双因素验证 (如启用):
  ├─ TOTP: totpAuthenticatorCheck
  └─ 备份码: 对称加密存储 (ENCRYPTION_KEY)
    ↓
创建 Session → 写入数据库 Session 表
    ↓
返回 sessionToken Cookie
```

**关键代码位置**：
- `apps/web/modules/auth/lib/authOptions.ts:189-363` - Credentials Provider
- `apps/web/lib/auth.ts` - 密码哈希与基础验证函数

#### 会话表结构 (Prisma Schema)

```prisma
model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique  // 存储在浏览器 Cookie 中
  userId       String
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  expires      DateTime // 会话硬过期时间
}
```

### 2.2 服务端会话获取

**核心函数**：`getServerSession(authOptions)`

```typescript
// 调用位置示例: apps/web/app/(app)/layout.tsx:19
const session = await getServerSession(authOptions);
const user = session?.user?.id ? await getUser(session.user.id) : null;
```

**Session 对象包含**：
```typescript
{
  user: {
    id: string;        // 用户ID (cuid)
    email: string;     // 用户邮箱
    isActive?: boolean; // 账号状态
  }
  expires: string;     // ISO 格式过期时间
}
```

---

## 三、成员角色体系 (Membership Roles)

### 3.1 组织级角色 (OrganizationRole)

| 角色 | 权限级别 | 主要权限 |
|------|---------|---------|
| `owner` | 最高 | 所有权限，包括删除组织、变更所有者 |
| `manager` | 高级 | 成员管理、项目管理、账单查看 |
| `billing` | 专项 | 仅账单相关操作 |
| `member` | 基础 | 需通过团队 (Team) 授予项目访问权 |

**Prisma Schema**:
```prisma
enum OrganizationRole {
  owner
  manager
  member
  billing
}

model Membership {
  organizationId String
  userId         String
  accepted       Boolean          @default(false)
  role           OrganizationRole @default(member)
  
  @@id([userId, organizationId])  // 复合主键
}
```

### 3.2 权限验证函数

**位置**：`apps/web/lib/organization/auth.ts`

```typescript
// 1. 检查用户是否属于该组织
export const canUserAccessOrganization = async (
  userId: string, 
  organizationId: string
): Promise<boolean>

// 2. 返回详细权限对象
export const verifyUserRoleAccess = async (
  organizationId: string,
  userId: string
): Promise<{
  hasCreateOrUpdateAccess: boolean;    // 创建/更新资源
  hasDeleteAccess: boolean;            // 删除资源
  hasCreateOrUpdateMembersAccess: boolean;  // 成员管理
  hasDeleteMembersAccess: boolean;            // 删除成员
  hasBillingAccess: boolean;           // 账单访问
}>
```

**权限判定逻辑**：
```typescript
// apps/web/lib/organization/auth.ts:40-52
if (!isOwner) {
  // 非 Owner 默认关闭所有权限
  accessObject.hasCreateOrUpdateAccess = false;
  accessObject.hasDeleteAccess = false;
  accessObject.hasCreateOrUpdateMembersAccess = false;
  accessObject.hasDeleteMembersAccess = false;
  accessObject.hasBillingAccess = false;
}

if (isManager) {
  // Manager 拥有成员管理和账单权限
  accessObject.hasCreateOrUpdateMembersAccess = true;
  accessObject.hasDeleteMembersAccess = true;
  accessObject.hasBillingAccess = true;
}
```

### 3.3 工具函数：权限标志提取

**位置**：`apps/web/lib/membership/utils.ts`

```typescript
export const getAccessFlags = (role?: OrganizationRole) => {
  const isOwner = role === "owner";
  const isManager = role === "manager";
  const isBilling = role === "billing";
  const isMember = role === "member";

  return { isManager, isOwner, isBilling, isMember };
};
```

---

## 四、环境访问控制 (Environment Access)

### 4.1 层级关系

```
Organization
    ├─ Project A
    │    ├─ Environment (production)
    │    └─ Environment (development)
    └─ Project B
         ├─ Environment (production)
         └─ Environment (development)
```

### 4.2 环境访问判定算法

**位置**：`apps/web/lib/environment/auth.ts:7-58`

```typescript
export const hasUserEnvironmentAccess = async (
  userId: string, 
  environmentId: string
) => {
  // 第一步：检查组织级直接访问权
  const orgMembership = await prisma.membership.findFirst({
    where: {
      userId,
      organization: {
        projects: {
          some: {
            environments: {
              some: { id: environmentId }
            }
          }
        }
      }
    }
  });

  // Owner / Manager / Billing 角色直接放行
  if (orgMembership?.role === "owner" || 
      orgMembership?.role === "manager" || 
      orgMembership?.role === "billing") {
    return true;
  }

  // 第二步：检查团队级项目访问权
  const teamMembership = await prisma.teamUser.findFirst({
    where: {
      userId,
      team: {
        projectTeams: {
          some: {
            project: {
              environments: {
                some: { id: environmentId }
              }
            }
          }
        }
      }
    }
  });

  return !!teamMembership;
};
```

**判定优先级**：
1. **组织角色优先**：Owner/Manager/Billing 角色自动获得所有环境访问权
2. **团队权限补充**：普通 Member 需通过所属团队 (Team) 获得项目访问权

### 4.3 团队与项目权限关联

```prisma
// 团队-用户关联
model TeamUser {
  teamId    String
  userId    String
  role      TeamUserRole  // admin / contributor
  @@id([teamId, userId])
}

// 团队-项目关联
model ProjectTeam {
  projectId  String
  teamId     String
  permission ProjectTeamPermission  // read / readWrite / manage
  @@id([projectId, teamId])
}
```

---

## 五、环境切换机制 (Environment Switching)

### 5.1 URL 驱动的上下文解析

**路由模式**：`/environments/[environmentId]/...`

```typescript
// 从 URL 路径参数提取 environmentId
// 位置: apps/web/app/(app)/environments/[environmentId]/layout.tsx

// 通过 environmentId 反向推导:
// 1. 获取 Environment → projectId
// 2. 获取 Project → organizationId  
// 3. 获取 Membership (userId + organizationId) → role
```

### 5.2 前端切换组件

**位置**：`apps/web/app/(app)/environments/[environmentId]/components/`

- **`ProjectAndOrgSwitch.tsx`** - 主切换组件
- **`OrganizationBreadcrumb.tsx`** - 组织切换面包屑
- **`ProjectBreadcrumb.tsx`** - 项目切换面包屑  
- **`EnvironmentBreadcrumb.tsx`** - 环境切换面包屑

```typescript
// 组件接收权限标志，控制 UI 显示
interface ProjectAndOrgSwitchProps {
  isOwnerOrManager: boolean;    // 控制组织/项目管理操作
  isMember: boolean;            // 控制只读视图
  isBilling: boolean;           // 控制账单入口
  isMembershipPending: boolean; // 待接受邀请状态
  // ...
}
```

### 5.3 开发环境特殊处理

```typescript
// development 环境显示环境面包屑
// production 环境隐藏 (apps/web/app/.../project-and-org-switch.tsx:44-45)
const currentEnvironment = environments.find((env) => env.id === currentEnvironmentId);
const showEnvironmentBreadcrumb = currentEnvironment?.type === "development";
```

---

## 六、关键安全机制

### 6.1 缓存策略

**Membership 查询缓存**：
```typescript
// apps/web/lib/membership/service.ts:45-47
// 使用 React Cache 进行请求级 deduplication
const getMembershipByUserIdOrganizationIdCached = reactCache(async (...));
```

### 6.2 防时序攻击

```typescript
// 密码验证使用 constant-time 比较
// apps/web/modules/auth/lib/authOptions.ts:234-235
const hashToVerify = user?.password || CONTROL_HASH;
const isValid = await verifyPassword(credentials.password, hashToVerify);
```

### 6.3 速率限制

```typescript
// 登录接口 IP 限流
// apps/web/modules/auth/lib/authOptions.ts:191
await applyIPRateLimit(rateLimitConfigs.auth.login);
```

---

## 七、数据模型关系图

```
┌─────────────────────────────────────────────────────────┐
│                         User                            │
│  id (cuid)                                              │
│  email (unique)                                         │
│  password (bcrypt hash)                                 │
│  twoFactorEnabled                                       │
└─────────────┬─────────────────────────┬─────────────────┘
              │                         │
              │ 1:N                     │ 1:N
              ▼                         ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│        Membership        │  │        TeamUser          │
│  userId + organizationId │  │  userId + teamId         │
│  role (owner/manager/...)│  │  role (admin/contributor)│
│  accepted (boolean)      │  └────────────┬─────────────┘
└─────────────┬────────────┘               │
              │ 1:1                         │
              ▼                             │
┌──────────────────────────┐               │
│      Organization        │               │
│  id                      │               │
│  name                    │               │
└─────────────┬────────────┘               │
              │ 1:N                         │
              ▼                             │
┌──────────────────────────┐               │
│          Team            │               │
│  id                      │               │
│  name                    │               │
└─────────────┬────────────┘               │
              │ 1:N                         │
              ▼                             │
┌──────────────────────────┐               │
│       ProjectTeam        │◄──────────────┘
│  projectId + teamId      │
│  permission (read/...)   │
└─────────────┬────────────┘
              │ 1:1
              ▼
┌──────────────────────────┐
│         Project          │
│  id                      │
│  name                    │
└─────────────┬────────────┘
              │ 1:N
              ▼
┌──────────────────────────┐
│       Environment        │
│  id                      │
│  type (dev/prod)         │
│  projectId               │
└──────────────────────────┘
```

---

## 八、权限判定完整流程总结

```
HTTP Request
    ↓
1.  Cookie (sessionToken) → Session 表查询
    ↓
2.  获取 userId (session.user.id)
    ↓
3.  URL 参数 environmentId → 推导 projectId → 推导 organizationId
    ↓
4.  Membership 表查询 (userId + organizationId) → role
    ↓
5.  权限分支：
    ├─ [owner/manager/billing] → 直接允许
    │
    └─ [member] → 检查 TeamUser → ProjectTeam 关联
           ↓
           ├─ 找到匹配 → 允许访问
           └─ 无匹配 → 拒绝访问 (403)
```

---

**文件索引**：

| 模块 | 路径 |
|------|------|
| 认证配置 | `apps/web/modules/auth/lib/authOptions.ts` |
| 组织权限 | `apps/web/lib/organization/auth.ts` |
| 环境权限 | `apps/web/lib/environment/auth.ts` |
| 成员服务 | `apps/web/lib/membership/service.ts` |
| 数据库 Schema | `packages/database/schema.prisma` |
| 切换组件 | `apps/web/app/(app)/environments/[environmentId]/components/` |
