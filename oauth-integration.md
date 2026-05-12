# 第三方 OAuth 集成技术架构文档

## 概述

Formbricks 实现了一套完整的第三方集成架构，支持将问卷采集的数据同步到 Notion、Airtable 和 Google Sheets 等平台。整个链路涵盖 OAuth 授权、资源选择、字段映射和数据同步四个核心阶段。

---

## 一、OAuth 授权流程

### 1.1 授权流程时序

```
用户点击授权按钮
    ↓
前端调用 GET /api/v1/integrations/{type} 获取授权URL
    ↓
前端跳转到第三方 OAuth 授权页面
    ↓
用户在第三方平台完成授权
    ↓
第三方平台回调到 /api/v1/integrations/{type}/callback
    ↓
后端使用 authorization_code 换取 access_token
    ↓
[平台差异] Notion 加密存储 / Airtable/Google Sheets 明文存储
    ↓
重定向回集成配置页面
```

---

### 1.2 Notion 授权实现（加密存储）

#### 授权端点 (`apps/web/app/api/v1/integrations/notion/route.ts`)

```typescript
export const GET = withV1ApiWrapper({
  handler: async ({ req, authentication }) => {
    const environmentId = req.headers.get("environmentId");

    // 构建 OAuth 授权 URL
    const authUrl = `${NOTION_AUTH_URL}&state=${environmentId}`;

    return { response: responses.successResponse({ authUrl }) };
  },
});
```

#### 回调处理 (`apps/web/app/api/v1/integrations/notion/callback/route.ts`)

```typescript
export const GET = withV1ApiWrapper({
  handler: async ({ req, authentication }) => {
    const queryParams = new URLSearchParams(url.split("?")[1]);
    const environmentId = queryParams.get("state");
    const code = queryParams.get("code");

    // 1. 使用 code 换取 access_token
    const response = await fetch("https://api.notion.com/v1/oauth/token", {
      method: "POST",
      headers: {
        Authorization: `Basic ${Buffer.from(
          `${NOTION_OAUTH_CLIENT_ID}:${NOTION_OAUTH_CLIENT_SECRET}`
        ).toString("base64")}`,
      },
      body: JSON.stringify({
        grant_type: "authorization_code",
        code,
        redirect_uri: NOTION_REDIRECT_URI,
      }),
    });

    const tokenData = await response.json();

    // 2. ⚠️ Notion 专属：AES-256-GCM 加密 access_token 后存储
    const encryptedAccessToken = symmetricEncrypt(
      tokenData.access_token,
      ENCRYPTION_KEY
    );
    tokenData.access_token = encryptedAccessToken;

    // 3. 存储到数据库
    await createOrUpdateIntegration(environmentId, {
      type: "notion",
      config: {
        key: tokenData,
        data: [],
      },
    });

    // 4. 重定向回集成页面
    return {
      response: Response.redirect(
        `${WEBAPP_URL}/environments/${environmentId}/workspace/integrations/notion`
      ),
    };
  },
});
```

#### 调用时解密 (`apps/web/lib/notion/service.ts`)

```typescript
const getHeaders = (config: TIntegrationNotionConfig) => {
  // 每次 API 调用前解密 token
  const decryptedToken = symmetricDecrypt(config.key.access_token, ENCRYPTION_KEY!);
  return {
    Accept: "application/json",
    "Content-Type": "application/json",
    Authorization: `Bearer ${decryptedToken}`,
    "Notion-Version": "2022-06-28",
  };
};
```

**Notion 授权特点：**
- 使用 Basic Auth 方式在 Token 请求中传递 Client ID 和 Secret
- Token 永不过期，无需刷新机制
- **⚠️ access_token 采用 AES-256-GCM 加密后存储**
- 每次 API 调用前动态解密

---

### 1.3 Airtable 授权实现（明文存储 + 自动刷新）

#### 授权端点 (`apps/web/app/api/v1/integrations/airtable/route.ts`)

```typescript
const scope = `data.records:read data.records:write schema.bases:read schema.bases:write user.email:read`;

export const GET = withV1ApiWrapper({
  handler: async ({ req, authentication }) => {
    const environmentId = req.headers.get("environmentId");

    // PKCE: 生成 code_verifier 和 code_challenge
    const codeVerifier = Buffer.from(
      environmentId + authentication.user.id + environmentId
    ).toString("base64");

    const codeChallenge = crypto
      .createHash("sha256")
      .update(codeVerifier)
      .digest("base64")
      .replace(/=/g, "")
      .replace(/\+/g, "-")
      .replace(/\//g, "_");

    // 构建授权 URL
    const authUrl = new URL("https://airtable.com/oauth2/v1/authorize");
    authUrl.searchParams.append("client_id", AIRTABLE_CLIENT_ID);
    authUrl.searchParams.append(
      "redirect_uri",
      WEBAPP_URL + "/api/v1/integrations/airtable/callback"
    );
    authUrl.searchParams.append("state", environmentId);
    authUrl.searchParams.append("scope", scope);
    authUrl.searchParams.append("response_type", "code");
    authUrl.searchParams.append("code_challenge_method", "S256");
    authUrl.searchParams.append("code_challenge", codeChallenge);

    return { response: responses.successResponse({ authUrl: authUrl.toString() }) };
  },
});
```

#### 回调处理 (`apps/web/app/api/v1/integrations/airtable/callback/route.ts`)

```typescript
export const GET = withV1ApiWrapper({
  handler: async ({ req, authentication }) => {
    const queryParams = new URLSearchParams(url.split("?")[1]);
    const environmentId = queryParams.get("state");
    const code = queryParams.get("code");

    // PKCE: 重新生成 code_verifier
    const code_verifier = Buffer.from(
      environmentId + authentication.user.id + environmentId
    ).toString("base64");

    // 1. 使用 code 换取 access_token
    const key = await fetchAirtableAuthToken({
      grant_type: "authorization_code",
      code,
      redirect_uri: WEBAPP_URL + "/api/v1/integrations/airtable/callback",
      client_id: AIRTABLE_CLIENT_ID,
      code_verifier,
    });

    // 2. ⚠️ Airtable：直接明文存储 token 数据
    //    key = { access_token, expiry_date, refresh_token }

    // 3. 存储到数据库（保留现有映射数据）
    const existingIntegration = await getIntegrationByType(environmentId, "airtable");
    await createOrUpdateIntegration(environmentId, {
      type: "airtable",
      config: {
        key,
        data: existingIntegration?.config?.data ?? [],
        email,
      },
    });

    return {
      response: Response.redirect(
        `${WEBAPP_URL}/environments/${environmentId}/workspace/integrations/airtable`
      ),
    };
  },
});
```

#### Token 自动刷新机制 (`apps/web/lib/airtable/service.ts`)

```typescript
export const fetchAirtableAuthToken = async (formData) => {
  const formBody = Object.keys(formData)
    .map((key) => `${encodeURIComponent(key)}=${encodeURIComponent(formData[key])}`)
    .join("&");

  const tokenReq = await fetch("https://airtable.com/oauth2/v1/token", {
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: formBody,
    method: "POST",
  });

  const tokenRes = await tokenReq.json();
  const { access_token, refresh_token, expires_in } = tokenRes;

  // 计算过期时间
  const expiry_date = new Date();
  expiry_date.setSeconds(expiry_date.getSeconds() + expires_in);

  return {
    access_token,
    expiry_date: expiry_date.toISOString(),
    refresh_token,
  };
};

// 每次调用前自动检查并刷新 Token
export const getAirtableToken = async (environmentId: string) => {
  const airtableIntegration = await getIntegrationByType(environmentId, "airtable");
  const { access_token, expiry_date, refresh_token } = airtableIntegration.config.key;

  const expiryDate = new Date(expiry_date);
  const currentDate = new Date();

  // Token 即将过期，使用 refresh_token 刷新
  if (currentDate >= expiryDate) {
    const newToken = await fetchAirtableAuthToken({
      grant_type: "refresh_token",
      refresh_token,
      client_id: AIRTABLE_CLIENT_ID,
    });

    // 更新数据库中的 Token（明文直接覆盖）
    await createOrUpdateIntegration(environmentId, {
      type: "airtable",
      config: {
        data: airtableIntegration.config.data ?? [],
        email: airtableIntegration.config.email ?? "",
        key: newToken,
      },
    });

    return newToken.access_token;
  }

  return access_token;
};
```

**Airtable 授权特点：**
- 使用 PKCE (Proof Key for Code Exchange) 增强安全性
- Token 有效期约 60 分钟，需实现刷新机制
- **⚠️ access_token、refresh_token、expiry_date 均以明文 JSON 格式存储**
- 每次 API 调用前自动检查并刷新过期 Token

---

### 1.4 Google Sheets 授权实现（明文存储 + 有效性检查）

#### 授权端点 (`apps/web/app/api/google-sheet/route.ts`)

```typescript
const scopes = [
  "https://www.googleapis.com/auth/spreadsheets",
  "https://www.googleapis.com/auth/userinfo.email",
];

export const GET = async (req: NextRequest) => {
  const environmentId = req.headers.get("environmentId");

  const oAuth2Client = new google.auth.OAuth2(
    GOOGLE_SHEETS_CLIENT_ID,
    GOOGLE_SHEETS_CLIENT_SECRET,
    GOOGLE_SHEETS_REDIRECT_URL
  );

  const authUrl = oAuth2Client.generateAuthUrl({
    access_type: "offline", // 必须指定以获取 refresh_token
    scope: scopes,
    prompt: "consent", // 强制显示同意页面，确保获取 refresh_token
    state: environmentId,
  });

  return responses.successResponse({ authUrl });
};
```

#### 回调处理 (`apps/web/app/api/google-sheet/callback/route.ts`)

```typescript
export const GET = async (req: Request) => {
  const url = new URL(req.url);
  const environmentId = url.searchParams.get("state");
  const code = url.searchParams.get("code");

  const oAuth2Client = new google.auth.OAuth2(
    GOOGLE_SHEETS_CLIENT_ID,
    GOOGLE_SHEETS_CLIENT_SECRET,
    GOOGLE_SHEETS_REDIRECT_URL
  );

  // 1. 换取 Token
  const token = await oAuth2Client.getToken(code);
  const key = token.res.data;
  // key = { access_token, refresh_token, expiry_date, scope, token_type, id_token }

  // 2. ⚠️ Google Sheets：直接明文存储完整 token 响应

  // 3. 存储到数据库
  const existingIntegration = await getIntegrationByType(environmentId, "googleSheets");
  await createOrUpdateIntegration(environmentId, {
    type: "googleSheets",
    config: {
      key,
      data: existingIntegration?.config?.data ?? [],
      email: userEmail,
    },
  });

  return Response.redirect(
    `${WEBAPP_URL}/environments/${environmentId}/workspace/integrations/google-sheets`
  );
};
```

#### Token 验证与刷新 (`apps/web/lib/googleSheet/service.ts`)

```typescript
const TOKEN_EXPIRY_BUFFER_MS = 5 * 60 * 1000; // 提前 5 分钟刷新
const GOOGLE_TOKENINFO_URL = "https://www.googleapis.com/oauth2/v1/tokeninfo";

// 验证 Access Token 是否有效（是否被用户撤销）
const isAccessTokenValid = async (accessToken: string): Promise<boolean> => {
  try {
    const res = await fetch(`${GOOGLE_TOKENINFO_URL}?access_token=${encodeURIComponent(accessToken)}`);
    return res.ok;
  } catch {
    return false;
  }
};

const authorize = async (googleSheetIntegrationData: TIntegrationGoogleSheets) => {
  const oAuth2Client = new google.auth.OAuth2(
    GOOGLE_SHEETS_CLIENT_ID,
    GOOGLE_SHEETS_CLIENT_SECRET,
    GOOGLE_SHEETS_REDIRECT_URL
  );
  const key = googleSheetIntegrationData.config.key;

  // 检查：Token 是否存在且未过期且未被撤销
  const hasStoredCredentials =
    key.access_token &&
    key.expiry_date &&
    key.expiry_date > Date.now() + TOKEN_EXPIRY_BUFFER_MS;

  if (hasStoredCredentials && (await isAccessTokenValid(key.access_token))) {
    oAuth2Client.setCredentials(key);
    return oAuth2Client;
  }

  // 使用 Refresh Token 刷新
  oAuth2Client.setCredentials({ refresh_token: key.refresh_token });

  try {
    const { credentials } = await oAuth2Client.refreshAccessToken();
    const mergedCredentials = {
      ...credentials,
      refresh_token: credentials.refresh_token ?? key.refresh_token, // 保留原有 refresh_token
    };

    // 更新数据库中的 Token（明文直接覆盖）
    await createOrUpdateIntegration(googleSheetIntegrationData.environmentId, {
      type: "googleSheets",
      config: {
        data: googleSheetIntegrationData.config.data ?? [],
        email: googleSheetIntegrationData.config.email ?? "",
        key: mergedCredentials,
      },
    });

    oAuth2Client.setCredentials(mergedCredentials);
    return oAuth2Client;
  } catch (error) {
    if (isInvalidGrantError(error)) {
      throw new AuthenticationError(GOOGLE_SHEET_INTEGRATION_INVALID_GRANT);
    }
    throw error;
  }
};
```

**Google Sheets 授权特点：**
- 使用官方 Google API Client Library
- `access_type: offline` 必须指定以获取 refresh_token
- `prompt: consent` 强制显示授权页面，确保每次重新授权都能获取 refresh_token
- **⚠️ 完整 token 响应（含 access_token、refresh_token 等）均以明文 JSON 格式存储**
- 每次 API 调用前检查 Token 有效性（是否被用户撤销）
- refresh_token 可能过期（用户 6 个月未使用时自动失效）

---

## 二、凭证存储策略对照表

| 平台 | access_token 存储 | refresh_token 存储 | 加密算法 | 使用前操作 |
|------|-------------------|---------------------|----------|------------|
| **Notion** | 加密存储 | 不适用（无刷新机制） | AES-256-GCM | 每次调用前 symmetricDecrypt 解密 |
| **Airtable** | 明文存储 | 明文存储 | - | 检查 expiry_date，过期自动刷新 |
| **Google Sheets** | 明文存储 | 明文存储 | - | 检查 expiry_date + 调用 tokeninfo 验证有效性，失效自动刷新 |

---

## 三、当前实现的安全边界与风险提示

### ⚠️ 已实现的安全措施

1. **Notion token 加密**：采用 AES-256-GCM 带认证加密，提供完整性校验
2. **PKCE 保护**：Airtable 使用 PKCE 防止授权码拦截攻击
3. **环境变量**：加密密钥 `ENCRYPTION_KEY` 通过环境变量注入，不随代码提交
4. **权限验证**：回调接口均验证用户对 Environment 的访问权限
5. **HTTPS**：生产环境强制 HTTPS，防止传输过程中的窃听

### ⚠️ 已知安全风险

1. **Airtable / Google Sheets token 明文存储**
   - 风险：数据库一旦泄露，攻击者可直接使用 access_token 操作用户的第三方平台数据
   - 影响范围：所有已连接 Airtable 或 Google Sheets 的环境
   - 建议：为这两个平台实现与 Notion 一致的对称加密

2. **refresh_token 明文存储**
   - 风险：refresh_token 生命周期较长（Google Sheets 可达 6 个月），泄露后可长期获取新的 access_token
   - 建议：对 refresh_token 单独加密存储，或实现 token 轮换机制

3. **PKCE code_verifier 可预测**
   - 当前实现：`Buffer.from(environmentId + userId + environmentId).toString("base64")`
   - 风险：攻击者若已知环境 ID 和用户 ID，可预测 code_verifier，降低 PKCE 的安全增益
   - 建议：改用 `crypto.randomBytes(32).toString("base64")` 生成不可预测的 verifier，并通过 state 参数或 session 传递

4. **state 参数仅包含 environmentId**
   - 风险：缺少 CSRF 防护，易受跨站请求伪造攻击
   - 建议：state 参数应包含随机生成的 nonce，并在回调时验证

5. **Token 撤销检测不完善**
   - 仅 Google Sheets 调用 tokeninfo 端点验证
   - Notion 和 Airtable 未检查 token 是否已被用户手动撤销
   - 建议：三平台统一实现调用前有效性检查

---

## 四、资源选择与映射

（后续章节保持原有内容，省略部分与之前一致）

### （以下为原有章节的延续）

## 字段同步策略

...

## 架构设计总结

...
