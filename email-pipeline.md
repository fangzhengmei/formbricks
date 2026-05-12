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

**证据**:
- `apps/web/modules/email/index.tsx:1` - `import { createTransport } from "nodemailer"`
- `apps/web/modules/email/index.tsx:23-36` - 所有 SMTP_* 常量导入
- `apps/web/modules/email/index.tsx:54` - `IS_SMTP_CONFIGURED = Boolean(SMTP_HOST && SMTP_PORT)`

---

### Resend Provider ❌ 未实现

**证据**:
- 整个代码库**不存在** `resend` 或 `Resend` 关键字的导入/调用（邮件模块内 0 匹配）
- 不存在 `import { Resend } from 'resend'` 或类似 SDK 引入
- 不存在任何 Resend API 调用代码
- `packages/email/package.json` 无 `resend` 依赖
- `apps/web/package.json` 无 `resend` 依赖

---

### 可切换 Provider 入口 ❌ 未实现

**证据**:
- 全代码库**不存在** `EMAIL_PROVIDER`、`emailProvider`、`MAIL_PROVIDER` 等配置变量
- `sendEmail` 函数（`index.tsx:71`）**只有** SMTP 单一路径实现
- 无 switch/case 或策略模式路由逻辑
- 无邮件 Provider 抽象接口或多实现结构
- `packages/email` 包仅导出模板渲染函数，不含任何发送适配层

---

## 1. 触发事件 (Trigger Events)

### 1.1 验证邮件触发

**位置**: `apps/web/modules/auth/verification-requested/actions.ts:53-83`

```typescript
export const resendVerificationEmailAction = actionClient.inputSchema(ZResendVerificationEmailAction).action(
  withAuditLogging("verificationEmailSent", "user", async ({ ctx, parsedInput }) => {
    await applyIPRateLimit(rateLimitConfigs.auth.verifyEmail); // 限流
    const user = await getUserByEmail(parsedInput.email);
    ctx.auditLoggingCtx.userId = user.id;
    await sendVerificationEmail({
      id: user.id,
      email: user.email,
      locale: user.locale,
      callbackUrl: validatedCallbackUrl,
      purpose,
    });
    return { success: true };
  })
);
```

**调用链**:
1. 用户在前端点击"重新发送验证邮件"
2. 触发 Server Action `resendVerificationEmailAction`
3. 执行 IP 限流检查 → 获取用户信息 → 审计日志 → 调用 `sendVerificationEmail`

**证据**:
- `apps/web/modules/auth/verification-requested/actions.ts:12` - `applyIPRateLimit` 导入
- `apps/web/modules/auth/verification-requested/actions.ts:16` - `sendVerificationEmail` 导入
- `apps/web/modules/auth/verification-requested/actions.ts:55` - 限流调用
- `apps/web/modules/auth/verification-requested/actions.ts:72-78` - 邮件发送调用

---

### 1.2 调查完成通知触发

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:160-290`

```typescript
if (event === "responseFinished") {
  // 查询所有开启通知的用户
  const usersWithNotifications = await prisma.user.findMany({
    where: {
      memberships: { some: { organization: { projects: { some: { environments: { some: { id: environmentId } } } } } } },
      notificationSettings: { path: ["alert", surveyId], equals: true },
    },
    select: { email: true, locale: true },
  });

  // 发送 follow-up 邮件
  if (survey.followUps?.length > 0) {
    const followUpsResult = await sendFollowUpsForResponse(response.id);
  }

  // 并发发送通知邮件
  const emailPromises = usersWithNotifications.map((user) =>
    sendResponseFinishedEmail(
      user.email,
      user.locale,
      environmentId,
      survey,
      response,
      responseCount
    ).catch((error) => {
      logger.error({ error, userEmail: user.email }, `Failed to send email to ${user.email}`);
    })
  );

  // 全部 settled 后才返回
  const results = await Promise.allSettled([...webhookPromises, ...emailPromises]);
}
```

**调用链**:
1. 调查响应完成 → 内部 cron/worker 调用 `POST /api/(internal)/pipeline`
2. 验证 CRON_SECRET 鉴权 → 查询 survey + organization
3. 查询所有开启该调查通知的团队成员
4. 并行发送 follow-up 邮件和通知邮件
5. 所有 Promise settled 后返回

**证据**:
- `apps/web/app/api/(internal)/pipeline/route.ts:23` - `sendResponseFinishedEmail` 导入
- `apps/web/app/api/(internal)/pipeline/route.ts:25` - `sendFollowUpsForResponse` 导入
- `apps/web/app/api/(internal)/pipeline/route.ts:173-224` - 查询开启通知的用户
- `apps/web/app/api/(internal)/pipeline/route.ts:237-251` - 批量发送邮件（带 .catch）
- `apps/web/app/api/(internal)/pipeline/route.ts:285` - `Promise.allSettled` 调用

---

### 1.3 Follow-up 跟进邮件触发

**位置**: `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:147-279`

```typescript
export const sendFollowUpsForResponse = async (
  responseId: string
): Promise<Result<FollowUpResult[], { code: FollowUpSendError; message: string }>> => {
  validateInputs([responseId, ZId]);
  const response = await getResponse(responseId);
  if (!response) return err({ code: FollowUpSendError.RESPONSE_NOT_FOUND, ... });

  const survey = await getSurvey(response.surveyId);
  if (!survey) return err({ code: FollowUpSendError.SURVEY_NOT_FOUND, ... });

  const organization = await getOrganizationByEnvironmentId(survey.environmentId);
  if (!organization) return err({ code: FollowUpSendError.ORG_NOT_FOUND, ... });

  // 1. 功能权限检查
  const surveyFollowUpsPermission = await getSurveyFollowUpsPermission(organization.id);
  if (!surveyFollowUpsPermission) return err({ code: FollowUpSendError.FOLLOW_UP_NOT_ALLOWED, ... });

  // 2. 组织级限流检查
  try {
    await applyRateLimit(rateLimitConfigs.actions.surveyFollowUp, organization.id);
  } catch {
    return err({ code: FollowUpSendError.RATE_LIMIT_EXCEEDED, ... });
  }

  // 3. 处理每个 follow-up 配置
  const followUpPromises = survey.followUps.map(async (followUp): Promise<FollowUpResult> => {
    const { trigger } = followUp;
    if (trigger.properties?.endingIds && !endingIds.includes(response.endingId)) {
      return { followUpId: followUp.id, status: "skipped" };
    }
    return evaluateFollowUp(followUp, survey, response, organization);
  });

  const followUpResults = await Promise.all(followUpPromises);
  return { ok: true, data: followUpResults };
};
```

**调用链**:
1. 由 pipeline 触发（或直接调用）
2. 查询响应、调查、组织数据 → 权限检查 → 限流检查
3. 遍历 follow-up 配置 → 检查 endingId 条件 → 解析收件人 → 发送邮件

**证据**:
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:14` - `applyRateLimit` 导入
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:185-191` - 权限检查
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:195-202` - 限流检查
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:213-230` - follow-up 循环处理

---

## 2. 模板渲染 (Template Rendering)

### 2.1 渲染函数列表

**位置**: `packages/email/src/lib/render.ts`

每个邮件类型对应独立的渲染函数，全部使用 `@react-email/render`：

```typescript
import { render } from "@react-email/render";

export async function renderVerificationEmail(props): Promise<string> {
  return await render(VerificationEmail(props));
}

export async function renderForgotPasswordEmail(props): Promise<string> {
  return await render(ForgotPasswordEmail(props));
}

export async function renderInviteEmail(props): Promise<string> {
  return await render(InviteEmail(props));
}

export async function renderResponseFinishedEmail(props): Promise<string> {
  return await render(ResponseFinishedEmail(props));
}

export async function renderFollowUpEmail(props): Promise<string> {
  return await render(FollowUpEmail(props));
}

// 其他渲染函数：renderNewEmailVerification, renderPasswordResetNotifyEmail,
// renderLinkSurveyEmail, renderEmbedSurveyPreviewEmail, renderEmailCustomizationPreviewEmail
```

**证据**:
- `packages/email/src/lib/render.ts:1` - `import { render } from "@react-email/render"`
- 所有 10 个 `render*Email` 函数全部使用同一 `render` 方法
- 返回类型均为 `Promise<string>`（HTML 字符串）

---

### 2.2 模板组件结构

**位置**: `packages/email/src/emails/` 下各子目录

```tsx
// 示例：VerificationEmail - packages/email/src/emails/auth/verification-email.tsx
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

**证据**:
- `packages/email/src/index.ts` - 导出全部 10 个邮件组件
- `packages/email/src/components/email-template.tsx` - 统一邮件布局组件
- 所有模板均接收 `t` 翻译函数和 `legalProps` 法律链接属性

---

### 2.3 国际化 (i18n) 注入 ✅ 已实现

**位置**: `apps/web/modules/email/index.tsx` 内所有 `send*Email` 函数

```typescript
// 示例：sendVerificationEmail
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

**证据**:
- `apps/web/modules/email/index.tsx:50` - `import { getTranslate } from "@/lingodotdev/server"`
- `apps/web/modules/email/index.tsx:115` - `sendVerificationNewEmail` 调用 `getTranslate(locale)`
- `apps/web/modules/email/index.tsx:146` - `sendVerificationEmail` 调用 `getTranslate(locale)`
- `apps/web/modules/email/index.tsx:183` - `sendPasswordResetLinkEmail` 调用 `getTranslate(user.locale)`
- 所有 10 个 `send*Email` 函数均在渲染前调用 `getTranslate(locale)` 并传入 `t` 函数

---

### 2.4 法律链接自动注入 ✅ 已实现

**位置**: `apps/web/modules/email/index.tsx:56-61`

```typescript
const legalProps: TEmailTemplateLegalProps = {
  privacyUrl: PRIVACY_URL || undefined,
  termsUrl: TERMS_URL || undefined,
  imprintUrl: IMPRINT_URL || undefined,
  imprintAddress: IMPRINT_ADDRESS || undefined,
};
```

**证据**:
- `apps/web/modules/email/index.tsx:26-39` - 四个法律链接相关常量导入
- `apps/web/modules/email/index.tsx:56-61` - `legalProps` 对象构建
- 所有 `render*Email` 调用均展开 `...legalProps` 传入模板

---

## 3. 发送适配 (Sending Adapter)

### 3.1 SMTP 发送流程 ✅ 已实现

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

**行为说明**:
- SMTP 配置缺失时**静默返回 false**，不抛出异常
- 发送失败时抛出 `InvalidInputError`，**向上冒泡**给调用者处理

**证据**:
- `apps/web/modules/email/index.tsx:72-75` - 配置检查逻辑
- `apps/web/modules/email/index.tsx:77-95` - transporter 创建
- `apps/web/modules/email/index.tsx:97-99` - 发件人默认值
- `apps/web/modules/email/index.tsx:100` - `transporter.sendMail()` 调用
- `apps/web/modules/email/index.tsx:103-106` - 错误抛出

---

### 3.2 各类型邮件发送入口列表 ✅ 已实现

| 邮件类型 | 入口函数 | 代码位置 |
|---------|---------|---------|
| 验证邮件 | `sendVerificationEmail` | `apps/web/modules/email/index.tsx:132-175` |
| 新邮箱验证 | `sendVerificationNewEmail` | `apps/web/modules/email/index.tsx:109-130` |
| 密码重置链接 | `sendPasswordResetLinkEmail` | `apps/web/modules/email/index.tsx:177-195` |
| 密码重置通知 | `sendPasswordResetNotifyEmail` | `apps/web/modules/email/index.tsx:197-208` |
| 团队邀请 | `sendInviteMemberEmail` | `apps/web/modules/email/index.tsx:210-229` |
| 邀请接受通知 | `sendInviteAcceptedEmail` | `apps/web/modules/email/index.tsx:231-244` |
| 调查完成通知 | `sendResponseFinishedEmail` | `apps/web/modules/email/index.tsx:246-306` |
| 嵌入预览邮件 | `sendEmbedSurveyPreviewEmail` | `apps/web/modules/email/index.tsx:308-330` |
| 白标定制预览 | `sendEmailCustomizationPreviewEmail` | `apps/web/modules/email/index.tsx:332-353` |
| 链接调查邮件 | `sendLinkSurveyToVerifiedEmail` | `apps/web/modules/email/index.tsx:355-378` |
| Follow-up 跟进邮件 | `sendFollowUpEmail` | `apps/web/modules/survey/follow-ups/lib/email.ts:19-137` |

**证据**:
- 以上 11 个函数均为真实存在的导出函数
- 每个函数内部均调用 `sendEmail` 执行最终发送

---

## 4. 失败处理 (Failure Handling)

### 4.1 Promise.allSettled 批量容错 ✅ 已实现

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:285`

```typescript
// Await webhook and email promises with allSettled to prevent early rejection
const results = await Promise.allSettled([...webhookPromises, ...emailPromises]);
results.forEach((result) => {
  if (result.status === "rejected") {
    logger.error({ error: result.reason, url: request.url }, "Promise rejected");
  }
});
```

**作用**: 在批量发送时，单个邮件发送失败**不会中断**整体流程，也不会导致 API 请求失败。

**证据**:
- `apps/web/app/api/(internal)/pipeline/route.ts:285` - `Promise.allSettled` 调用
- `apps/web/app/api/(internal)/pipeline/route.ts:286-289` - 失败日志记录

---

### 4.2 单个 Promise .catch() 日志记录 ✅ 已实现

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:237-251`

```typescript
const emailPromises = usersWithNotifications.map((user) =>
  sendResponseFinishedEmail(
    user.email,
    user.locale,
    environmentId,
    survey,
    response,
    responseCount
  ).catch((error) => {
    logger.error(
      { error, url: request.url, userEmail: user.email },
      `Failed to send email to ${user.email}`
    );
  })
);
```

**作用**: 捕获单个邮件发送异常，记录详细日志（含接收人邮箱），但**不进行任何重试或补偿**。

**证据**:
- `apps/web/app/api/(internal)/pipeline/route.ts:245-250` - `.catch()` 回调逻辑
- 日志包含 `error`、`userEmail`、`url` 等上下文字段

---

### 4.3 Follow-up 邮件的 Result 模式 ✅ 已实现

**位置**: `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:20-140`

```typescript
const evaluateFollowUp = async (...): Promise<FollowUpResult> => {
  try {
    await sendFollowUpEmail(...);
    return { followUpId: followUp.id, status: "success" as const };
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

**证据**:
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:20-140` - `evaluateFollowUp` 函数
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:235-251` - 错误结果收集与日志

---

### 4.4 SMTP 配置缺失时静默跳过 ✅ 已实现

**位置**: `apps/web/modules/email/index.tsx:72-75`

```typescript
if (!IS_SMTP_CONFIGURED) {
  logger.info("SMTP is not configured, skipping email sending");
  return false;
}
```

**作用**: 未配置邮件服务的环境中，不抛出异常导致业务流程中断。

**证据**:
- `apps/web/modules/email/index.tsx:72-75` - 配置检查与静默返回逻辑

---

### 4.5 自动失败重试 ❌ 未实现

**证据**:
- 全代码库**不存在** `withRetry`、`retry`、`retryAsync`、`attemptRetry` 等重试函数
- 无 `p-retry`、`@lifeomic/attempt`、`async-retry` 等重试库的导入
- `sendEmail` 函数（`index.tsx:71`）异常直接抛出，无循环重试逻辑
- 无 `maxRetries`、`retryDelay`、`exponentialBackoff` 等重试配置常量
- **所有** `send*Email` 函数调用后均无重试包装

**注**：仅有的"重发"是业务层面手动触发（非失败自动重试）：
- `resendInvite`（组织邀请）：用户点击"重新发送"按钮触发，非失败自动重试
- `resendVerificationEmailAction`（验证邮件）：用户点击触发，非失败自动重试

---

### 4.6 死信队列/补偿机制 ❌ 未实现

**证据**:
- 无邮件发送状态持久化表（无 `email_logs`、`email_queue` 等表定义）
- 无补偿任务调度逻辑
- 无死信队列处理代码

---

## 5. 限流 (Rate Limiting)

### 5.1 限流核心实现 ✅ 已实现

**算法**: **固定时间窗口计数**（Fixed Window Counter）

**位置**: `apps/web/modules/core/rate-limit/rate-limit.ts:13-135`

```typescript
export const checkRateLimit = async (
  config: TRateLimitConfig,
  identifier: string
): Promise<Result<TRateLimitResponse, string>> => {
  const now = Date.now();

  // 关键：按时间分桶计算窗口起始时间（固定窗口）
  const windowStart = Math.floor(now / (config.interval * 1000)) * config.interval;

  // Redis key 包含 namespace + identifier + windowStart（每个窗口独立计数）
  const key = createCacheKey.rateLimit.core(config.namespace, identifier, windowStart);

  const windowEnd = windowStart + config.interval;
  const ttlSeconds = Math.max(1, Math.ceil((windowEnd * 1000 - now) / 1000));

  // Lua 脚本保证原子性
  const luaScript = `
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local ttl = tonumber(ARGV[2])
    
    -- 原子递增计数器
    local current = redis.call('INCR', key)
    
    -- 仅在第一次递增时设置过期时间（避免窗口延长）
    if current == 1 then
      redis.call('EXPIRE', key, ttl)
    end
    
    -- 返回当前计数 + 是否超限
    return {current, current <= limit and 1 or 0}
  `;

  const result = (await redis.eval(luaScript, {
    keys: [key],
    arguments: [config.allowedPerInterval.toString(), ttlSeconds.toString()],
  })) as [number, number];
  const [currentCount, isAllowed] = result;

  return ok({
    allowed: isAllowed === 1,
    retryAfter: isAllowed === 1 ? undefined : ttlSeconds,
  });
};
```

**算法特征说明**:
- **窗口类型**: 固定时间窗口（非滑动窗口）
- **计数方式**: 每个时间窗口使用独立的 Redis key（嵌入 `windowStart`）
- **原子性**: Lua 脚本保证 `INCR` + `EXPIRE` 原子操作，防止多 Pod 竞态
- **窗口边界**: `windowStart = floor(timestamp / interval) * interval`，与滑动窗口算法有本质区别

**关键特性**:
- **失效开放**: Redis 不可用时跳过限流，保证系统可用性
- **可全局禁用**: 通过 `RATE_LIMITING_DISABLED` 环境变量关闭
- **违规监控**: 限流违规自动记录 Sentry breadcrumb

**证据**:
- `apps/web/modules/core/rate-limit/rate-limit.ts:36` - `windowStart` 固定分桶计算
- `apps/web/modules/core/rate-limit/rate-limit.ts:37` - key 嵌入 `windowStart`
- `apps/web/modules/core/rate-limit/rate-limit.ts:46-61` - Lua 脚本仅执行 INCR + 条件 EXPIRE
- 无滑动窗口所需的历史窗口清理、多窗口加权计算等逻辑

---

### 5.2 限流配置 ✅ 已实现

**位置**: `apps/web/modules/core/rate-limit/rate-limit-configs.ts`

```typescript
export const rateLimitConfigs = {
  auth: {
    login: { interval: 900, allowedPerInterval: 10, namespace: "auth:login" },
    signup: { interval: 3600, allowedPerInterval: 30, namespace: "auth:signup" },
    forgotPassword: { interval: 3600, allowedPerInterval: 5, namespace: "auth:forgot" },
    verifyEmail: { interval: 3600, allowedPerInterval: 10, namespace: "auth:verify" },
  },
  api: {
    v1: { interval: 60, allowedPerInterval: 100, namespace: "api:v1" },
    v2: { interval: 60, allowedPerInterval: 100, namespace: "api:v2" },
    v3: { interval: 60, allowedPerInterval: 100, namespace: "api:v3" },
    client: { interval: 60, allowedPerInterval: 100, namespace: "api:client" },
  },
  actions: {
    emailUpdate: { interval: 3600, allowedPerInterval: 3, namespace: "action:email" },
    accountDeletion: { interval: 3600, allowedPerInterval: 5, namespace: "action:account-delete" },
    surveyFollowUp: { interval: 3600, allowedPerInterval: 50, namespace: "action:followup" },
    sendLinkSurveyEmail: { interval: 3600, allowedPerInterval: 10, namespace: "action:send-link-survey-email" },
    licenseRecheck: { interval: 60, allowedPerInterval: 5, namespace: "action:license-recheck" },
  },
  storage: {
    upload: { interval: 60, allowedPerInterval: 5, namespace: "storage:upload" },
    delete: { interval: 60, allowedPerInterval: 5, namespace: "storage:delete" },
  },
} as const;
```

**邮件相关限流项**:
- `auth.verifyEmail`: 10 次/小时
- `auth.forgotPassword`: 5 次/小时
- `actions.surveyFollowUp`: 50 次/小时
- `actions.sendLinkSurveyEmail`: 10 次/小时

**证据**:
- `apps/web/modules/core/rate-limit/rate-limit-configs.ts` - 完整配置文件
- 以上 4 项为邮件发送相关的限流配置

---

### 5.3 IP 级限流 (验证邮件/密码重置) ✅ 已实现

**位置**: `apps/web/modules/core/rate-limit/helpers.ts:57-60`

```typescript
export const applyIPRateLimit = async (config: TRateLimitConfig): Promise<TRateLimitResponse> => {
  const identifier = await getClientIdentifier(); // 获取并哈希客户端 IP
  return await applyRateLimit(config, identifier);
};
```

**调用示例**:
```typescript
// apps/web/modules/auth/verification-requested/actions.ts:55
await applyIPRateLimit(rateLimitConfigs.auth.verifyEmail);
```

**证据**:
- `apps/web/modules/core/rate-limit/helpers.ts:15-25` - `getClientIdentifier` 函数（IP 哈希）
- `apps/web/modules/core/rate-limit/helpers.ts:57-60` - `applyIPRateLimit` 实现
- `apps/web/modules/auth/verification-requested/actions.ts:55` - 验证邮件限流调用

---

### 5.4 组织级限流 (Follow-up 邮件) ✅ 已实现

**位置**: `apps/web/modules/core/rate-limit/helpers.ts:34-48`

```typescript
export const applyRateLimit = async (
  config: TRateLimitConfig,
  identifier: string
): Promise<TRateLimitResponse> => {
  const result = await checkRateLimit(config, identifier);
  if (!result.ok || !result.data.allowed) {
    throw new TooManyRequestsError("Maximum number of requests reached...", retryAfter);
  }
  return result.data;
};
```

**调用示例**:
```typescript
// apps/web/modules/survey/follow-ups/lib/follow-ups.ts:196
await applyRateLimit(rateLimitConfigs.actions.surveyFollowUp, organization.id);
```

**证据**:
- `apps/web/modules/core/rate-limit/helpers.ts:34-48` - `applyRateLimit` 通用实现
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts:196` - 使用 `organization.id` 作为 identifier
- 限流触发时抛出 `TooManyRequestsError`

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

## 7. 状态总览表

| 能力 | 状态 | 证据位置 |
|-----|------|---------|
| **SMTP Provider** | ✅ 已实现 | `apps/web/modules/email/index.tsx:71-107` |
| **Resend Provider** | ❌ 未实现 | 全库无 Resend SDK/API 调用，邮件模块 0 匹配 |
| **Provider 切换入口** | ❌ 未实现 | 无 EMAIL_PROVIDER 配置，无路由逻辑 |
| **React Email 模板渲染** | ✅ 已实现 | `packages/email/src/lib/render.ts:1` |
| **i18n 国际化注入** | ✅ 已实现 | 所有 `send*Email` 函数均调用 `getTranslate(locale)` |
| **法律链接自动注入** | ✅ 已实现 | `apps/web/modules/email/index.tsx:56-61` |
| **固定时间窗口限流** | ✅ 已实现 | `apps/web/modules/core/rate-limit/rate-limit.ts:36` |
| **Redis Lua 原子操作** | ✅ 已实现 | `apps/web/modules/core/rate-limit/rate-limit.ts:46-61` |
| **IP 级限流** | ✅ 已实现 | `rate-limit/helpers.ts:57-60` + 验证邮件调用 |
| **组织级限流** | ✅ 已实现 | `rate-limit/helpers.ts:34-48` + follow-up 调用 |
| **批量发送容错 (allSettled)** | ✅ 已实现 | `pipeline/route.ts:285` |
| **单个发送 .catch() 日志** | ✅ 已实现 | `pipeline/route.ts:245-250` |
| **Result 模式错误收集** | ✅ 已实现 | `follow-ups/lib/follow-ups.ts:133-139` |
| **SMTP 配置缺失静默跳过** | ✅ 已实现 | `apps/web/modules/email/index.tsx:72-75` |
| **自动失败重试** | ❌ 未实现 | 全库无重试函数，无重试库导入 |
| **指数退避** | ❌ 未实现 | 无相关配置或代码 |
| **死信队列/补偿机制** | ❌ 未实现 | 无邮件发送状态持久化表 |
| **邮件发送状态持久化** | ❌ 未实现 | 无 email_logs 表定义 |

---

## 附录: 核心文件位置

| 文件路径 | 说明 |
|---------|------|
| `apps/web/modules/email/index.tsx` | 邮件发送核心模块 |
| `packages/email/src/lib/render.ts` | 模板渲染函数 |
| `packages/email/src/emails/` | 各类型邮件 React 组件 |
| `apps/web/modules/core/rate-limit/rate-limit.ts` | 限流核心实现（固定窗口算法） |
| `apps/web/modules/core/rate-limit/rate-limit-configs.ts` | 限流配置 |
| `apps/web/modules/core/rate-limit/helpers.ts` | 限流辅助函数 |
| `apps/web/app/api/(internal)/pipeline/route.ts` | 调查完成触发入口 |
| `apps/web/modules/survey/follow-ups/lib/follow-ups.ts` | Follow-up 邮件入口 |
| `apps/web/modules/auth/verification-requested/actions.ts` | 验证邮件触发入口 |
