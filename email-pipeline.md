# Formbricks 邮件发送管道架构

## 概述

Formbricks 的邮件系统是一个三层架构的邮件发送管道，负责向受访者、团队成员和管理员发送各类通知邮件。系统设计支持灵活的邮件 provider 切换、模板化渲染以及可靠的失败处理机制。

---

## 1. 模板渲染层 (Template Rendering Layer)

### 位置
- **模板定义**: `packages/email/src/emails/`
- **渲染引擎**: `packages/email/src/lib/render.ts`
- **调用入口**: `apps/web/modules/email/index.tsx`

### 核心设计

#### 1.1 React 组件化模板
所有邮件模板使用 React 组件定义，支持 TypeScript 类型安全：

```typescript
// 模板示例：VerificationEmail
export const VerificationEmail = (props: VerificationEmailProps) => {
  const { t, verifyLink, verificationRequestLink, ...legalProps } = props;
  return (
    <EmailTemplate {...legalProps}>
      <Heading>{t("emails.verify_your_email")}</Heading>
      <Button href={verifyLink}>{t("emails.verify_email")}</Button>
    </EmailTemplate>
  );
};
```

#### 1.2 统一渲染函数
`render.ts` 提供类型安全的渲染函数，每个邮件类型对应一个渲染函数：

```typescript
// packages/email/src/lib/render.ts
export async function renderVerificationEmail(
  props: {
    verifyLink: string;
    verificationRequestLink: string;
    t: TFunction;
  } & TEmailTemplateLegalProps
): Promise<string> {
  return await render(VerificationEmail(props));
}

// 类似的渲染函数：
// - renderForgotPasswordEmail
// - renderInviteEmail
// - renderResponseFinishedEmail
// - renderFollowUpEmail
// ...
```

#### 1.3 多语言支持
- 使用 `@/lingodotdev/server` 的 `getTranslate` 函数获取翻译实例
- 翻译键统一使用 `emails.*` 命名空间（如 `emails.verify_your_email`）
- 支持用户级别的 locale 配置

#### 1.4 法律链接注入
所有模板自动注入法律链接：
- `privacyUrl`: 隐私政策链接
- `termsUrl`: 服务条款链接  
- `imprintUrl`: 印鉴链接
- `imprintAddress`: 印鉴地址

这些配置从环境变量读取，统一注入每个邮件模板。

---

## 2. Provider 适配层 (Provider Adaptation Layer)

### 位置
- **核心实现**: `apps/web/modules/email/index.tsx`
- **配置常量**: `apps/web/lib/constants.ts`

### 2.1 当前 Provider: SMTP (Nodemailer)

系统当前使用 Nodemailer 作为邮件发送库，支持完整的 SMTP 配置：

```typescript
export const IS_SMTP_CONFIGURED = Boolean(SMTP_HOST && SMTP_PORT);

export const sendEmail = async (emailData: SendEmailDataProps): Promise<boolean> => {
  if (!IS_SMTP_CONFIGURED) {
    logger.info("SMTP is not configured, skipping email sending");
    return false;
  }
  
  const transporter = createTransport({
    host: SMTP_HOST,
    port: SMTP_PORT,
    secure: SMTP_SECURE_ENABLED,
    ...(SMTP_AUTHENTICATED ? {
      auth: {
        type: "LOGIN",
        user: SMTP_USER,
        pass: SMTP_PASSWORD,
      }
    } : {}),
    tls: {
      rejectUnauthorized: SMTP_REJECT_UNAUTHORIZED_TLS,
    },
    logger: DEBUG,
    debug: DEBUG,
  } as SMTPTransport.Options);

  const emailDefaults = {
    from: `${MAIL_FROM_NAME ?? "Formbricks"} <${MAIL_FROM ?? "noreply@formbricks.com"}>`,
  };
  
  await transporter.sendMail({ ...emailDefaults, ...emailData });
  return true;
};
```

### 2.2 环境变量配置

| 变量名 | 说明 |
|--------|------|
| `SMTP_HOST` | SMTP 服务器地址 |
| `SMTP_PORT` | SMTP 端口 |
| `SMTP_SECURE_ENABLED` | 是否启用 TLS |
| `SMTP_AUTHENTICATED` | 是否需要认证 |
| `SMTP_USER` / `SMTP_PASSWORD` | SMTP 凭据 |
| `MAIL_FROM` / `MAIL_FROM_NAME` | 发件人配置 |

### 2.3 Provider 扩展设计

当前架构已为多 Provider 支持预留扩展点：

**未来接入 Resend 的建议实现：**

```typescript
// apps/web/modules/email/providers/resend.ts
import { Resend } from 'resend';

export const sendViaResend = async (emailData) => {
  const resend = new Resend(process.env.RESEND_API_KEY);
  return resend.emails.send({
    from: emailData.from,
    to: emailData.to,
    subject: emailData.subject,
    html: emailData.html,
  });
};

// 在 index.tsx 中实现 Provider 路由
export const sendEmail = async (emailData) => {
  const provider = EMAIL_PROVIDER || 'smtp'; // 'resend' | 'smtp'
  
  switch (provider) {
    case 'resend':
      return sendViaResend(emailData);
    case 'smtp':
    default:
      return sendViaSmtp(emailData);
  }
};
```

---

## 3. 失败重试与限流 (Retry & Rate Limiting)

### 3.1 限流机制 (Rate Limiting)

#### 位置
- **配置**: `apps/web/modules/core/rate-limit/rate-limit-configs.ts`
- **核心逻辑**: `apps/web/modules/core/rate-limit/rate-limit.ts`
- **辅助函数**: `apps/web/modules/core/rate-limit/helpers.ts`

#### 限流配置

```typescript
export const rateLimitConfigs = {
  auth: {
    verifyEmail: { 
      interval: 3600,        // 1小时
      allowedPerInterval: 10, // 10次
      namespace: "auth:verify" 
    },
  },
  actions: {
    surveyFollowUp: { 
      interval: 3600, 
      allowedPerInterval: 50, 
      namespace: "action:followup" 
    },
    sendLinkSurveyEmail: {
      interval: 3600,
      allowedPerInterval: 10,
      namespace: "action:send-link-survey-email",
    },
  },
};
```

#### 实现特点
- **滑动窗口算法**: 使用 Redis 实现精确的滑动窗口限流
- **多层命名空间**: 按功能模块划分不同的限流桶
- **组织级隔离**: 后续邮件支持按组织 ID 进行限流隔离
- **失败即阻止**: 触发限流后立即阻止后续请求，返回友好错误信息

#### 应用示例

```typescript
// 验证邮件限流
await applyIPRateLimit(rateLimitConfigs.auth.verifyEmail);

// Follow-up 邮件限流（按组织）
await applyRateLimit(rateLimitConfigs.actions.surveyFollowUp, organization.id);
```

### 3.2 失败处理机制

#### 策略 1: Promise.allSettled 容错
在批量邮件发送（如 Pipeline 中的响应完成通知）时，使用 `Promise.allSettled` 确保单个邮件失败不影响整体流程：

```typescript
// apps/web/app/api/(internal)/pipeline/route.ts
const emailPromises = usersWithNotifications.map((user) =>
  sendResponseFinishedEmail(...).catch((error) => {
    logger.error({ error, userEmail: user.email }, `Failed to send email to ${user.email}`);
  })
);

const results = await Promise.allSettled([...webhookPromises, ...emailPromises]);
results.forEach((result) => {
  if (result.status === "rejected") {
    logger.error({ error: result.reason }, "Promise rejected");
  }
});
```

#### 策略 2: 详细的错误日志
每个邮件发送函数都包含详细的错误捕获和日志记录：

```typescript
export const sendVerificationEmail = async ({...}) => {
  try {
    // 发送逻辑...
  } catch (error) {
    logger.error(error, "Error in sendVerificationEmail");
    throw error; // 向上抛出让调用者处理
  }
};
```

#### 策略 3: Follow-up 邮件的 Result 模式
对于跟进邮件，使用 Result 模式进行精细的错误处理：

```typescript
// apps/web/modules/survey/follow-ups/lib/follow-ups.ts
const evaluateFollowUp = async (...): Promise<FollowUpResult> => {
  try {
    await sendFollowUpEmail(...);
    return { followUpId: followUp.id, status: "success" };
  } catch (error) {
    return {
      followUpId: followUp.id,
      status: "error",
      error: error instanceof Error ? error.message : "Something went wrong",
    };
  }
};
```

### 3.3 重试机制 (当前状态与建议)

**当前实现状态**:
- 目前**没有内置的自动重试机制**
- 依赖于调用方的手动重试（如用户点击"重新发送"按钮）
- `resendInvite` 函数是手动重试的一个示例

**建议的重试实现方案**:

```typescript
// 建议添加到 apps/web/modules/email/lib/retry.ts
const withRetry = async <T>(
  fn: () => Promise<T>,
  maxRetries: number = 3,
  delayMs: number = 1000
): Promise<T> => {
  let lastError;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      logger.warn(`Email attempt ${i + 1} failed, retrying...`);
      await new Promise(resolve => setTimeout(resolve, delayMs * Math.pow(2, i))); // 指数退避
    }
  }
  
  throw lastError;
};

// 使用方式
export const sendEmailWithRetry = async (emailData) => {
  return withRetry(() => sendEmail(emailData), 3);
};
```

---

## 4. 完整邮件流程链路

### 4.1 以"验证邮件"为例的完整流程

```
用户注册/登录
    ↓
调用 resendVerificationEmailAction
    ↓
[限流检查] applyIPRateLimit(rateLimitConfigs.auth.verifyEmail)
    ↓
获取用户信息 + 生成 JWT token
    ↓
构建验证链接 verifyLink & verificationRequestLink
    ↓
[模板渲染] renderVerificationEmail({ t, verifyLink, ...legalProps })
    ↓
[Provider 发送] sendEmail({ to, subject, html })
    ↓
    ├─ SMTP 配置检查
    ├─ 创建 Nodemailer transporter
    └─ 调用 transporter.sendMail()
    ↓
返回发送结果
```

### 4.2 以"调查完成通知邮件"为例的批量流程

```
Response Finished 事件触发
    ↓
进入 /api/(internal)/pipeline
    ↓
查询所有开启通知的团队成员/管理员
    ↓
并发发送邮件 (Promise.allSettled + .catch())
    │
    ├─ 每个邮件:
    │   ├─ 获取用户 locale 的翻译实例
    │   ├─ 处理响应数据（图片 URL 转换）
    │   ├─ renderResponseFinishedEmail
    │   └─ sendEmail
    │
    └─ 失败的邮件仅记录日志，不影响其他
    ↓
所有邮件处理完成后返回
```

### 4.3 以"Follow-up 跟进邮件"为例的条件触发流程

```
用户完成调查
    ↓
sendFollowUpsForResponse(responseId)
    ↓
[权限检查] 组织是否有 follow-up 功能权限
    ↓
[限流检查] applyRateLimit(..., organization.id)
    ↓
遍历 survey.followUps 配置
    ↓
[条件判断] 检查 endingId 是否匹配 trigger 配置
    ↓
解析收件人（直接邮箱 / 响应数据中的邮箱）
    ↓
[内容处理] sanitizeHtml + parseRecallInfo
    ↓
[模板渲染] renderFollowUpEmail
    ↓
[Provider 发送] sendEmail
    ↓
收集所有结果并记录错误日志
```

---

## 5. 邮件类型总览

| 邮件类型 | 触发场景 | 接收者 | 限流配置 |
|---------|---------|-------|---------|
| 验证邮件 | 注册、邮箱变更 | 用户 | auth.verifyEmail |
| 密码重置邮件 | 忘记密码 | 用户 | auth.forgotPassword |
| 邀请邮件 | 团队邀请 | 受邀人 | - |
| 邀请接受通知 | 受邀人接受邀请 | 邀请人 | - |
| 调查完成通知 | 收到新响应 | 团队成员/管理员 | - |
| 链接调查邮件 | 手动发送给受访者 | 受访者 | actions.sendLinkSurveyEmail |
| Follow-up 邮件 | 满足结束条件时触发 | 动态收件人 | actions.surveyFollowUp |
| 嵌入预览邮件 | 模板预览测试 | 测试邮箱 | - |
| 白标定制预览 | 定制设置预览 | 管理员 | - |

---

## 6. 架构优化建议

### 6.1 引入消息队列
当前邮件发送是**同步阻塞**的，建议引入 BullMQ 或类似队列系统：
- 解耦邮件发送与业务逻辑
- 支持异步后台处理
- 内置重试、延迟、优先级等功能

### 6.2 邮件发送状态持久化
添加 `email_logs` 表记录每封邮件：
- 发送状态（pending/sent/failed）
- 发送时间/重试次数
- 错误信息
- 邮件元数据（类型、接收者）

### 6.3 Provider 故障转移
实现多 Provider 自动故障转移：
```
Resend 发送失败 → 自动降级到 SMTP → 记录告警
```

### 6.4 统一邮件日志与监控
- 结构化日志（包含 messageId、provider、duration 等）
- 邮件发送成功率监控
- 投递延迟告警

---

## 附录: 核心文件位置

| 文件路径 | 说明 |
|---------|------|
| `packages/email/src/index.ts` | 邮件包导出入口 |
| `packages/email/src/lib/render.ts` | 模板渲染函数 |
| `apps/web/modules/email/index.tsx` | 邮件发送核心模块 |
| `apps/web/modules/core/rate-limit/` | 限流模块 |
| `apps/web/app/api/(internal)/pipeline/route.ts` | 批量邮件触发点 |
| `apps/web/modules/survey/follow-ups/lib/email.ts` | Follow-up 邮件逻辑 |
