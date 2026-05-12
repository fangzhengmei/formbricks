# Formbricks 邮件发送管道架构

---

## 0. Provider 与可切换入口状态确认

### 默认 Provider: SMTP (Nodemailer) ✅ 已实现

**位置**: `apps/web/modules/email/index.tsx:71-107`

```typescript
export const sendEmail = async (emailData: SendEmailDataProps): Promise<boolean> => {
  if (!IS_SMTP_CONFIGURED) {
    logger.info("SMTP is not configured, skipping email sending");
    return false;
  }
  const transporter = createTransport({
    host: SMTP_HOST,
    port: SMTP_PORT,
    secure: SMTP_SECURE_ENABLED,
    auth: SMTP_AUTHENTICATED ? { type: "LOGIN", user: SMTP_USER, pass: SMTP_PASSWORD } : {},
    tls: { rejectUnauthorized: SMTP_REJECT_UNAUTHORIZED_TLS },
  });
  await transporter.sendMail({ from: MAIL_FROM, ...emailData });
  return true;
};
```

**配置项** (环境变量):
- `SMTP_HOST`, `SMTP_PORT` - SMTP 服务器配置
- `SMTP_SECURE_ENABLED`, `SMTP_AUTHENTICATED` - TLS/认证开关
- `SMTP_USER`, `SMTP_PASSWORD` - 凭据
- `MAIL_FROM`, `MAIL_FROM_NAME` - 发件人

### 可切换 Provider 入口 ❌ 未实现

**证据**:
- 代码库中**不存在** `EMAIL_PROVIDER` 或类似配置变量
- 代码库中**不存在** Resend SDK 导入或 Resend API 调用
- `sendEmail` 函数**只有** SMTP 实现路径，无 switch/case 或策略模式路由
- 无邮件 Provider 抽象层或接口定义

---

## 1. 触发事件 (Trigger Events)

### 1.1 验证邮件触发

**位置**: `apps/web/modules/auth/verification-requested/actions.ts:53-83`

```typescript
export const resendVerificationEmailAction = actionClient.action(
  withAuditLogging(async ({ ctx, parsedInput }) => {
    await applyIPRateLimit(rateLimitConfigs.auth.verifyEmail); // 限流
    const user = await getUserByEmail(parsedInput.email);
    ctx.auditLoggingCtx.userId = user.id;
    await sendVerificationEmail({ id: user.id, email: user.email, locale: user.locale });
    return { success: true };
  })
);
```

**调用链**:
1. 用户在前端点击"重新发送验证邮件"
2. 触发 Server Action `resendVerificationEmailAction`
3. 执行 IP 限流检查 → 获取用户信息 → 审计日志 → 调用 `sendVerificationEmail`

---

### 1.2 调查完成通知触发

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:160-290`

```typescript
if (event === "responseFinished") {
  // 查询所有开启通知的用户
  const usersWithNotifications = await prisma.user.findMany({
    where: { notificationSettings: { path: ["alert", surveyId], equals: true } },
    select: { email: true, locale: true },
  });

  // 发送 follow-up 邮件
  if (survey.followUps?.length > 0) {
    const followUpsResult = await sendFollowUpsForResponse(response.id);
  }

  // 并发发送通知邮件
  const emailPromises = usersWithNotifications.map((user) =>
    sendResponseFinishedEmail(...).catch(logger.error)
  );

  // 全部 settled 后才返回
  await Promise.allSettled([...webhookPromises, ...emailPromises]);
}
```

**调用链**:
1. 调查响应完成 → 内部 cron/worker 调用 `POST /api/(internal)/pipeline`
2. 验证 CRON_SECRET 鉴权 → 查询 survey + organization
3. 查询所有开启该调查通知的团队成员
4. 并行发送 follow-up 邮件和通知邮件
5. 所有 Promise settled 后返回

---

### 1.3 Follow-up 跟进邮件触发

**位置**: `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:147-279`

```typescript
export const sendFollowUpsForResponse = async (responseId: string) => {
  const response = await getResponse(responseId);
  const survey = await getSurvey(response.surveyId);
  const organization = await getOrganizationByEnvironmentId(survey.environmentId);

  // 1. 功能权限检查
  const surveyFollowUpsPermission = await getSurveyFollowUpsPermission(organization.id);
  if (!surveyFollowUpsPermission) return err(FOLLOW_UP_NOT_ALLOWED);

  // 2. 组织级限流检查
  await applyRateLimit(rateLimitConfigs.actions.surveyFollowUp, organization.id);

  // 3. 处理每个 follow-up 配置
  const followUpPromises = survey.followUps.map(async (followUp) => {
    // 检查 endingId 条件匹配
    if (trigger.properties?.endingIds && !endingIds.includes(response.endingId)) {
      return { status: "skipped" };
    }
    return evaluateFollowUp(followUp, survey, response, organization);
  });

  return Promise.all(followUpPromises);
};
```

**调用链**:
1. 由 pipeline 触发（或直接调用）
2. 查询响应、调查、组织数据 → 权限检查 → 限流检查
3. 遍历 follow-up 配置 → 检查 endingId 条件 → 解析收件人 → 发送邮件

---

## 2. 模板渲染 (Template Rendering)

### 2.1 渲染函数列表

**位置**: `packages/email/src/lib/render.ts`

每个邮件类型对应独立的渲染函数：

```typescript
export async function renderVerificationEmail(props) {
  return await render(VerificationEmail(props));  // @react-email/render
}

export async function renderForgotPasswordEmail(props) {
  return await render(ForgotPasswordEmail(props));
}

export async function renderInviteEmail(props) {
  return await render(InviteEmail(props));
}

export async function renderResponseFinishedEmail(props) {
  return await render(ResponseFinishedEmail(props));
}

export async function renderFollowUpEmail(props) {
  return await render(FollowUpEmail(props));
}

// 其他：renderNewEmailVerification, renderPasswordResetNotifyEmail,
// renderLinkSurveyEmail, renderEmbedSurveyPreviewEmail, renderEmailCustomizationPreviewEmail
```

---

### 2.2 模板组件结构

**位置**: `packages/email/src/emails/` 下各子目录

```tsx
// 示例：VerificationEmail
export const VerificationEmail = ({ t, verifyLink, verificationRequestLink, ...legalProps }) => (
  <EmailTemplate {...legalProps}>
    <EmailHeader>
      <Heading>{t("emails.verify_your_email")}</Heading>
    </EmailHeader>
    <EmailBody>
      <Text>{t("emails.click_below_to_verify")}</Text>
      <EmailButton href={verifyLink}>{t("emails.verify_email")}</EmailButton>
      <Text>{t("emails.trouble_clicking_link")}</Text>
      <Link href={verificationRequestLink}>{verificationRequestLink}</Link>
    </EmailBody>
  </EmailTemplate>
);
```

---

### 2.3 国际化 (i18n) 注入

**位置**: `apps/web/modules/email/index.tsx:146`

```typescript
export const sendVerificationEmail = async ({ locale, ... }) => {
  const t = await getTranslate(locale);  // 获取对应语言的翻译实例
  const html = await renderVerificationEmail({
    t,  // 翻译函数传入模板
    verifyLink,
    verificationRequestLink,
    ...legalProps,
  });
  return await sendEmail({ to: email, subject: t("emails.verification_email_subject"), html });
};
```

---

### 2.4 法律链接自动注入

**位置**: `apps/web/modules/email/index.tsx:56-61`

```typescript
const legalProps: TEmailTemplateLegalProps = {
  privacyUrl: PRIVACY_URL || undefined,
  termsUrl: TERMS_URL || undefined,
  imprintUrl: IMPRINT_URL || undefined,
  imprintAddress: IMPRINT_ADDRESS || undefined,
};
```

所有邮件模板自动接收 `legalProps`，在页脚展示法律链接。

---

## 3. 发送适配 (Sending Adapter)

### 3.1 SMTP 发送流程

**位置**: `apps/web/modules/email/index.tsx:71-107`

```typescript
export const sendEmail = async (emailData: SendEmailDataProps): Promise<boolean> => {
  // 1. 配置检查 - 未配置 SMTP 则静默跳过
  if (!IS_SMTP_CONFIGURED) {
    logger.info("SMTP is not configured, skipping email sending");
    return false;
  }

  try {
    // 2. 创建 Nodemailer transporter
    const transporter = createTransport({
      host: SMTP_HOST,
      port: SMTP_PORT,
      secure: SMTP_SECURE_ENABLED,
      ...(SMTP_AUTHENTICATED ? { auth: { type: "LOGIN", user: SMTP_USER, pass: SMTP_PASSWORD } } : {}),
      tls: { rejectUnauthorized: SMTP_REJECT_UNAUTHORIZED_TLS },
      logger: DEBUG,
      debug: DEBUG,
    } as SMTPTransport.Options);

    // 3. 默认发件人配置
    const emailDefaults = {
      from: `${MAIL_FROM_NAME ?? "Formbricks"} <${MAIL_FROM ?? "noreply@formbricks.com"}>`,
    };

    // 4. 发送邮件
    await transporter.sendMail({ ...emailDefaults, ...emailData });
    return true;
  } catch (error) {
    logger.error(error, "Error in sendEmail");
    throw new InvalidInputError("Incorrect SMTP credentials");
  }
};
```

**注意事项**:
- SMTP 配置缺失时**静默返回 false**，不抛出异常
- 发送失败时抛出 `InvalidInputError`，**向上冒泡**给调用者处理

---

### 3.2 各类型邮件发送入口

| 邮件类型 | 入口函数 | 位置 |
|---------|---------|------|
| 验证邮件 | `sendVerificationEmail` | index.tsx:132-175 |
| 新邮箱验证 | `sendVerificationNewEmail` | index.tsx:109-130 |
| 密码重置 | `sendPasswordResetLinkEmail` | index.tsx:177-195 |
| 密码重置通知 | `sendPasswordResetNotifyEmail` | index.tsx:197-208 |
| 团队邀请 | `sendInviteMemberEmail` | index.tsx:210-229 |
| 邀请接受通知 | `sendInviteAcceptedEmail` | index.tsx:231-244 |
| 调查完成通知 | `sendResponseFinishedEmail` | index.tsx:246-306 |
| 预览邮件 | `sendEmbedSurveyPreviewEmail` | index.tsx:308-330 |
| 白标定制预览 | `sendEmailCustomizationPreviewEmail` | index.tsx:332-353 |
| 链接调查邮件 | `sendLinkSurveyToVerifiedEmail` | index.tsx:355-378 |
| Follow-up 邮件 | `sendFollowUpEmail` | follow-ups/lib/email.ts:19-137 |

---

## 4. 失败处理 (Failure Handling)

### 4.1 失败处理状态: 部分实现 ✅

**已实现的失败处理策略**:

#### 策略 1: Promise.allSettled 容错
**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:285`

```typescript
// Await webhook and email promises with allSettled to prevent early rejection
const results = await Promise.allSettled([...webhookPromises, ...emailPromises]);
results.forEach((result) => {
  if (result.status === "rejected") {
    logger.error({ error: result.reason }, "Promise rejected");
  }
});
```

**作用**: 在批量发送时，单个邮件发送失败**不会中断**整体流程，也不会导致 API 请求失败。

---

#### 策略 2: 单个 Promise .catch() 日志记录
**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:237-251`

```typescript
const emailPromises = usersWithNotifications.map((user) =>
  sendResponseFinishedEmail(...).catch((error) => {
    logger.error({ error, userEmail: user.email }, `Failed to send email to ${user.email}`);
  })
);
```

**作用**: 捕获单个邮件发送异常，记录详细日志（含接收人邮箱），但**不进行任何重试或补偿**。

---

#### 策略 3: Follow-up 邮件的 Result 模式
**位置**: `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:20-140`

```typescript
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

**作用**:
- 单个 follow-up 失败**不影响**其他 follow-up 的发送
- 失败结果被收集并批量记录到日志
- 但**没有重试机制**，失败即终结

---

#### 策略 4: SMTP 配置缺失时静默跳过
**位置**: `apps/web/modules/email/index.tsx:72-75`

```typescript
if (!IS_SMTP_CONFIGURED) {
  logger.info("SMTP is not configured, skipping email sending");
  return false;
}
```

**作用**: 未配置邮件服务的环境中，不抛出异常导致业务流程中断。

---

### 4.2 自动重试状态: 未实现 ❌

**证据**:
- 代码库中**不存在** `withRetry`、`retry`、`backoff` 等重试相关函数
- 无 `p-retry`、`@lifeomic/attempt` 等重试库的导入
- `sendEmail` 异常直接抛出，无循环重试逻辑
- 无任何指数退避、最大重试次数等配置

**仅有的"重发"机制（非自动重试）**:
- `resendInvite`（`organization/settings/teams/lib/invite.ts:38`）是用户主动触发的"重新发送邀请"，属于业务层面的手动重发，**不是自动失败重试**
- `resendVerificationEmailAction` 是用户点击"重新发送验证邮件"触发，**不是失败自动重试**

---

## 5. 限流 (Rate Limiting)

### 5.1 限流状态: 已实现 ✅

**核心实现**: Redis + Lua 脚本的滑动窗口限流

**位置**: `apps/web/modules/core/rate-limit/rate-limit.ts:13-135`

```typescript
export const checkRateLimit = async (config: TRateLimitConfig, identifier: string) => {
  const now = Date.now();
  const windowStart = Math.floor(now / (config.interval * 1000)) * config.interval;
  const key = createCacheKey.rateLimit.core(config.namespace, identifier, windowStart);

  // Lua 脚本保证原子性，防止多 Pod 环境竞态
  const luaScript = `
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local ttl = tonumber(ARGV[2])
    
    local current = redis.call('INCR', key)
    if current == 1 then redis.call('EXPIRE', key, ttl) end
    return {current, current <= limit and 1 or 0}
  `;

  const result = await redis.eval(luaScript, { keys: [key], arguments: [...] });
  const [currentCount, isAllowed] = result;

  return {
    allowed: isAllowed === 1,
    retryAfter: isAllowed === 1 ? undefined : ttlSeconds,
  };
};
```

**关键特性**:
- **原子性**: Lua 脚本保证 INCR + EXPIRE 原子操作，避免多 Pod 竞态
- **失效开放**: Redis 不可用时跳过限流，保证系统可用性
- **可全局禁用**: 通过 `RATE_LIMITING_DISABLED` 环境变量关闭
- **违规监控**: 限流违规自动记录 Sentry breadcrumb

---

### 5.2 限流配置

**位置**: `apps/web/modules/core/rate-limit/rate-limit-configs.ts`

```typescript
export const rateLimitConfigs = {
  auth: {
    verifyEmail: { interval: 3600, allowedPerInterval: 10, namespace: "auth:verify" },
    forgotPassword: { interval: 3600, allowedPerInterval: 5, namespace: "auth:forgot" },
  },
  actions: {
    surveyFollowUp: { interval: 3600, allowedPerInterval: 50, namespace: "action:followup" },
    sendLinkSurveyEmail: { interval: 3600, allowedPerInterval: 10, namespace: "action:send-link-survey-email" },
  },
};
```

---

### 5.3 限流应用场景

#### 场景 1: IP 级限流 (验证邮件)
**位置**: `apps/web/modules/auth/verification-requested/actions.ts:55`

```typescript
await applyIPRateLimit(rateLimitConfigs.auth.verifyEmail);
```

`applyIPRateLimit` 内部使用客户端 IP 作为 identifier。

---

#### 场景 2: 组织级限流 (Follow-up 邮件)
**位置**: `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:196`

```typescript
await applyRateLimit(rateLimitConfigs.actions.surveyFollowUp, organization.id);
```

使用 `organization.id` 作为 identifier，按组织进行限流隔离。

---

### 5.4 限流辅助函数

**位置**: `apps/web/modules/core/rate-limit/helpers.ts`

```typescript
// IP 级限流
export const applyIPRateLimit = async (config: TRateLimitConfig) => {
  const ip = await getIp();  // 从 headers 获取客户端 IP
  const result = await checkRateLimit(config, ip);
  if (!result.ok || !result.data.allowed) {
    throw new TooManyRequestsError();
  }
  return result.data;
};

// 自定义 identifier 限流
export const applyRateLimit = async (config: TRateLimitConfig, identifier: string) => {
  const result = await checkRateLimit(config, identifier);
  if (!result.ok || !result.data.allowed) {
    throw new TooManyRequestsError();
  }
  return result.data;
};
```

**限流触发行为**: 抛出 `TooManyRequestsError`，由上层 Server Action 处理为 HTTP 429 响应。

---

## 6. 三种典型调用链汇总

### 链 1: 验证邮件 (用户触发 + IP 限流)

```
用户点击"重新发送验证邮件"
    ↓
Server Action: resendVerificationEmailAction
    ↓
[限流] applyIPRateLimit(auth.verifyEmail)
    ↓ 未超限
获取用户信息 getUserByEmail()
    ↓
[审计日志] withAuditLogging
    ↓
sendVerificationEmail()
    ├─ getTranslate(user.locale) → i18n
    ├─ 创建 token + verification links
    ├─ renderVerificationEmail(t, links, legalProps) → HTML
    └─ sendEmail() → Nodemailer SMTP
    ↓
返回 { success: true }
```

---

### 链 2: 调查完成通知 (系统触发 + 批量容错)

```
用户完成调查 → Response 落库
    ↓
内部 Cron/Worker 调用 POST /api/(internal)/pipeline
    ↓
[鉴权] 验证 x-api-key == CRON_SECRET
    ↓
查询 survey + organization
    ↓
查询所有开启通知的团队成员 (notificationSettings)
    ↓
[异步] sendFollowUpsForResponse(responseId)
    ↓
并发映射发送通知邮件
    ├─ Promise.allSettled 包裹
    ├─ 每个 Promise 带 .catch(logger.error)
    └─ 单个失败不影响整体
    ↓
等待所有 settled
    ↓
返回 200 OK
```

---

### 链 3: Follow-up 跟进邮件 (条件触发 + 组织限流)

```
sendFollowUpsForResponse(responseId)
    ↓
查询 response → survey → organization
    ↓
[权限] getSurveyFollowUpsPermission(organization.id)
    ↓ 有权限
[限流] applyRateLimit(actions.surveyFollowUp, organization.id)
    ↓ 未超限
遍历 survey.followUps[]
    ↓
[条件判断] endingId 匹配检查
    ├─ 不匹配 → { status: "skipped" }
    └─ 匹配 → evaluateFollowUp()
        ├─ 解析收件人邮箱
        ├─ sendFollowUpEmail()
        └─ try/catch 包裹返回 { status: "success" | "error" }
    ↓
收集所有结果 → 记录错误日志
    ↓
返回 Result<FollowUpResult[]>
```

---

## 7. 状态总览

| 能力 | 状态 | 证据位置 |
|-----|------|---------|
| **SMTP Provider** | ✅ 已实现 | `apps/web/modules/email/index.tsx:71-107` |
| **Resend Provider** | ❌ 未实现 | 全库无 Resend SDK/API 调用 |
| **Provider 切换入口** | ❌ 未实现 | 无 EMAIL_PROVIDER 配置/路由逻辑 |
| **React Email 模板渲染** | ✅ 已实现 | `packages/email/src/lib/render.ts` |
| **i18n 国际化** | ✅ 已实现 | 所有 `send*Email` 函数均调用 `getTranslate(locale)` |
| **IP 级限流** | ✅ 已实现 | `rate-limit/helpers.ts: applyIPRateLimit()` |
| **组织级限流** | ✅ 已实现 | `rate-limit/helpers.ts: applyRateLimit()` |
| **Redis 原子性限流** | ✅ 已实现 | `rate-limit/rate-limit.ts: Lua 脚本` |
| **批量发送容错 (allSettled)** | ✅ 已实现 | `pipeline/route.ts:285` |
| **单个发送 .catch() 日志** | ✅ 已实现 | `pipeline/route.ts:245` |
| **Result 模式错误收集** | ✅ 已实现 | `follow-ups/lib/follow-ups.ts:133-139` |
| **自动失败重试** | ❌ 未实现 | 全库无重试逻辑 |
| **指数退避** | ❌ 未实现 | 无相关代码 |
| **死信队列/补偿机制** | ❌ 未实现 | 无相关代码 |
| **邮件发送状态持久化** | ❌ 未实现 | 无 email_logs 表或持久化逻辑 |

---

## 附录: 核心文件位置

| 文件路径 | 说明 |
|---------|------|
| `apps/web/modules/email/index.tsx` | 邮件发送核心模块 |
| `packages/email/src/lib/render.ts` | 模板渲染函数 |
| `packages/email/src/emails/` | 各类型邮件 React 组件 |
| `apps/web/modules/core/rate-limit/rate-limit.ts` | 限流核心实现 |
| `apps/web/modules/core/rate-limit/rate-limit-configs.ts` | 限流配置 |
| `apps/web/modules/core/rate-limit/helpers.ts` | 限流辅助函数 |
| `apps/web/app/api/(internal)/pipeline/route.ts` | 调查完成触发入口 |
| `apps/web/modules/survey/follow-ups/lib/follow-ups.ts` | Follow-up 邮件入口 |
| `apps/web/modules/auth/verification-requested/actions.ts` | 验证邮件触发入口 |
