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

    // 3. 获取用户邮箱
    const email = await getEmail(key.access_token);

    // 4. 存储到数据库（保留现有映射数据）
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

  const session = await getServerSession(authOptions);

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

  // 3. 获取用户邮箱
  oAuth2Client.setCredentials({ access_token: key.access_token });
  const oauth2 = google.oauth2({ auth: oAuth2Client, version: "v2" });
  const userInfo = await oauth2.userinfo.get();
  const userEmail = userInfo.data.email;

  // 4. 存储到数据库
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

### 4.1 数据模型设计

#### 集成基础类型 (`packages/types/integration/shared-types.ts`)

```typescript
interface ZIntegrationBase {
  id: string;
  environmentId: string;
}

interface ZIntegrationBaseSurveyData {
  createdAt: Date;
  elementIds: string[];           // 要同步的问卷元素ID
  includeVariables: boolean;      // 是否包含变量
  includeHiddenFields: boolean;   // 是否包含隐藏字段
  includeMetadata: boolean;       // 是否包含元数据
  includeCreatedAt: boolean;      // 是否包含创建时间
  elements: string;               // 元素名称描述
  surveyId: string;               // 关联问卷ID
  surveyName: string;             // 问卷名称
}
```

#### Notion 特定配置

```typescript
interface TIntegrationNotionConfigData {
  mapping: Array<{
    element: { id: string; name: string; type: string };  // Formbricks 字段
    column: { id: string; name: string; type: string };   // Notion 字段
  }>;
  databaseId: string;    // Notion 数据库ID
  databaseName: string;  // Notion 数据库名称
  surveyId: string;      // 关联的问卷ID
  surveyName: string;    // 问卷名称
}
```

#### Airtable 特定配置

```typescript
interface TIntegrationAirtableConfigData {
  baseId: string;       // Airtable Base ID
  tableId: string;      // Airtable Table ID
  tableName: string;    // Table 名称
  surveyId: string;     // 关联的问卷ID
  surveyName: string;   // 问卷名称
  elementIds: string[]; // 选中的问题ID列表
  elements: string;     // 问题描述（全部问题/已选问题）
  includeVariables: boolean;
  includeHiddenFields: boolean;
  includeMetadata: boolean;
  includeCreatedAt: boolean;
}
```

#### Google Sheets 特定配置

```typescript
interface TIntegrationGoogleSheetsConfigData {
  spreadsheetId: string;     // Google Sheets 电子表格ID
  spreadsheetName: string;   // 表格名称
  surveyId: string;          // 关联的问卷ID
  surveyName: string;        // 问卷名称
  elementIds: string[];      // 选中的问题ID列表
  elements: string;          // 问题描述
  includeVariables: boolean;
  includeHiddenFields: boolean;
  includeMetadata: boolean;
  includeCreatedAt: boolean;
}
```

---

### 4.2 Notion 资源选择流程

#### 步骤1：获取数据库列表

```typescript
// apps/web/lib/notion/service.ts
export const getNotionDatabases = async (environmentId: string) => {
  const notionIntegration = await getIntegrationByType(environmentId, "notion");
  if (notionIntegration && notionIntegration.config?.key.bot_id) {
    const res = await fetch("https://api.notion.com/v1/search", {
      headers: getHeaders(notionIntegration.config),
      method: "POST",
      body: JSON.stringify({
        page_size: 100,
        filter: { value: "database", property: "object" },
      }),
    });
    return (await res.json()).results;
  }
  return [];
};
```

#### 步骤2：前端字段映射界面

```typescript
// apps/web/app/(app)/environments/[environmentId]/workspace/integrations/notion/components/AddIntegrationModal.tsx

// 1. 选择目标数据库
<DropdownSelector
  label="选择 Notion 数据库"
  items={databases}
  selectedItem={selectedDatabase}
  setSelectedItem={setSelectedDatabase}
/>

// 2. 选择要同步的问卷
<DropdownSelector
  label="选择问卷"
  items={surveys}
  selectedItem={selectedSurvey}
  setSelectedItem={setSelectedSurvey}
/>

// 3. 显示数据库现有字段，供双向映射
const dbItems = Object.keys(selectedDatabase.properties).map(fieldKey => ({
  id: selectedDatabase.properties[fieldKey].id,
  name: selectedDatabase.properties[fieldKey].name,
  type: selectedDatabase.properties[fieldKey].type
}));

// 4. MappingRow 组件实现双向映射
{selectedDatabase && selectedSurvey && (
  <div>
    <Label>映射 Formbricks 字段到 Notion 属性</Label>
    {mapping.map((m, idx) => (
      <MappingRow
        key={m.id}
        idx={idx}
        mapping={mapping}
        setMapping={setMapping}
        filteredElementItems={getFilteredElementItems(idx)}
        dbItems={dbItems}
        elementItems={elementItems}
      />
    ))}
  </div>
)}
```

**Notion 映射特点：**
- 精确字段级别的双向映射
- 用户需预先在 Notion 数据库中创建目标字段
- 类型兼容性检查（如日期→日期）
- 至少需要配置一组映射

---

### 4.3 Airtable 资源选择流程

#### 步骤1：获取 Base 列表

```typescript
// apps/web/lib/airtable/service.ts
export const getBases = async (accessToken: string) => {
  const req = await fetch("https://api.airtable.com/v0/meta/bases", {
    headers: { Authorization: `Bearer ${accessToken}` },
  });
  const res = await req.json();
  return res.bases;
};
```

#### 步骤2：获取 Base 下的 Table 列表

```typescript
// apps/web/app/api/v1/integrations/airtable/tables/route.ts
export const GET = withV1ApiWrapper({
  handler: async ({ req, authentication }) => {
    // 1. 用户鉴权检查
    if (!authentication || !("user" in authentication)) {
      return { response: responses.notAuthenticatedResponse() };
    }

    // 2. environmentId 来自请求头，而非 query 参数
    const url = req.url;
    const environmentId = req.headers.get("environmentId");

    // 3. baseId 使用 Zod 进行参数校验
    const queryParams = new URLSearchParams(url.split("?")[1]);
    const baseId = z.string().safeParse(queryParams.get("baseId"));

    if (!baseId.success) {
      return {
        response: responses.badRequestResponse("Base Id is Required"),
      };
    }

    if (!environmentId) {
      return {
        response: responses.badRequestResponse("environmentId is missing"),
      };
    }

    // 4. 用户环境权限校验
    const canUserAccessEnvironment = await hasUserEnvironmentAccess(
      authentication.user.id,
      environmentId
    );
    if (!canUserAccessEnvironment) {
      return {
        response: responses.unauthorizedResponse(),
      };
    }

    // 5. 检查集成是否存在
    const integration = await getIntegrationByType(environmentId, "airtable");
    if (!integration) {
      return {
        response: responses.notFoundResponse("Integration not found", environmentId),
      };
    }

    // 6. 使用 getAirtableToken 确保 token 已自动刷新
    const freshAccessToken = await getAirtableToken(environmentId);
    const tables = await getTables(
      { ...integration.config.key, access_token: freshAccessToken },
      baseId.data
    );

    return { response: responses.successResponse(tables) };
  },
});

// apps/web/lib/airtable/service.ts - 完整调用链
const tableFetcher = async (key: TIntegrationAirtableCredential, baseId: string) => {
  const req = await fetch(`https://api.airtable.com/v0/meta/bases/${baseId}/tables`, {
    headers: { Authorization: `Bearer ${key.access_token}` },
  });

  if (!req.ok) {
    const body = await req.text().catch(() => "");
    throw new Error(`Airtable API error fetching tables: ${req.status} ${req.statusText} ${body}`);
  }

  // 返回原始响应，不直接提取 tables 数组
  const res = await req.json();
  return res;
};

// getTables 通过 Zod schema 校验后返回包装结构
export const getTables = async (key: TIntegrationAirtableCredential, baseId: string) => {
  const res = await tableFetcher(key, baseId);
  // ZIntegrationAirtableTables.parse 返回的是 { tables: [...] } 结构
  return ZIntegrationAirtableTables.parse(res);
};
```

#### 步骤3：前端资源选择界面

```typescript
// apps/web/app/(app)/environments/[environmentId]/workspace/integrations/airtable/components/AddIntegrationModal.tsx

// 1. 选择 Base
<BaseSelectDropdown
  control={control}
  isLoading={isLoading}
  fetchTable={fetchTable}
  airtableArray={airtableArray}
  setValue={setValue}
/>

// 2. 选择 Table
<Controller
  control={control}
  name="table"
  render={({ field }) => (
    <Select onValueChange={field.onChange}>
      <SelectTrigger />
      <SelectContent>
        {tables.map((item) => (
          <SelectItem key={item.id} value={item.id}>
            {item.name}
          </SelectItem>
        ))}
      </SelectContent>
    </Select>
  )}
/>

// 3. 选择问卷 + 勾选问题复选框
<DropdownSelector
  label="选择问卷"
  items={surveys}
  selectedItem={selectedSurvey}
  setSelectedItem={setSelectedSurvey}
/>

{elements.map((element) => (
  <ElementCheckbox
    element={element}
    selectedSurvey={selectedSurvey}
    field={field}
  />
))}

// 4. 附加选项开关
<AdditionalIntegrationSettings
  includeVariables={includeVariables}
  setIncludeVariables={setIncludeVariables}
  includeHiddenFields={includeHiddenFields}
  includeMetadata={includeMetadata}
  setIncludeHiddenFields={setIncludeHiddenFields}
  setIncludeMetadata={setIncludeMetadata}
  includeCreatedAt={includeCreatedAt}
  setIncludeCreatedAt={setIncludeCreatedAt}
/>
```

**Airtable 映射特点：**
- 问题标题自动作为列名
- 自动创建缺失字段（统一为 singleLineText 类型）
- 支持变量、隐藏字段、元数据、创建时间的附加同步
- 无需预先在 Airtable 中创建字段

---

### 4.4 Google Sheets 资源选择流程

#### 步骤1：用户输入电子表格 URL

```typescript
// 从 URL 中提取 spreadsheetId
export const extractSpreadsheetIdFromUrl = (url: string): string => {
  const match = url.match(/\/d\/([a-zA-Z0-9-_]+)/);
  if (!match) throw new Error("Invalid Google Sheets URL");
  return match[1];
};

// 验证 spreadsheetId 是否有效且用户有访问权限
export const getSpreadsheetNameById = async (
  googleSheetIntegrationData: TIntegrationGoogleSheets,
  spreadsheetId: string
): Promise<string> => {
  const authClient = await authorize(googleSheetIntegrationData);
  const sheets = google.sheets({ version: "v4", auth: authClient });

  return new Promise((resolve, reject) => {
    sheets.spreadsheets.get({ spreadsheetId }, (err, response) => {
      if (err) {
        const msg = err.message.toLowerCase();
        if (msg.includes("permission") || msg.includes("caller does not have")) {
          reject(new OperationNotAllowedError("Insufficient permission"));
        } else {
          reject(err);
        }
        return;
      }
      resolve(response.data.properties.title);
    });
  });
};
```

#### 步骤2：前端资源选择界面

```typescript
// apps/web/app/(app)/environments/[environmentId]/workspace/integrations/google-sheets/components/AddIntegrationModal.tsx

// 1. 输入 Google Sheets URL
<Input
  value={spreadsheetUrl}
  onChange={(e) => setSpreadsheetUrl(e.target.value)}
  placeholder="https://docs.google.com/spreadsheets/d/<your-spreadsheet-id>"
/>

// 2. 验证 URL 有效性
const isValid = isValidGoogleSheetsUrl(spreadsheetUrl);
if (!isValid) {
  throw new Error("Please enter a valid spreadsheet URL");
}

// 3. 验证用户对表格的访问权限（写入前检查）
const spreadsheetName = await getSpreadsheetNameByIdAction({
  googleSheetIntegration,
  environmentId,
  spreadsheetId,
});

// 4. 选择问卷 + 勾选问题（同 Airtable 模式）
<DropdownSelector
  label="选择问卷"
  items={surveys}
  selectedItem={selectedSurvey}
  setSelectedItem={setSelectedSurvey}
/>

{elements.map((element) => (
  <ElementCheckbox
    element={element}
    selectedSurvey={selectedSurvey}
    field={field}
  />
))}

// 5. 附加选项开关
<AdditionalIntegrationSettings
  includeVariables={includeVariables}
  setIncludeVariables={setIncludeVariables}
  includeHiddenFields={includeHiddenFields}
  includeMetadata={includeMetadata}
  setIncludeHiddenFields={setIncludeHiddenFields}
  setIncludeMetadata={setIncludeMetadata}
  includeCreatedAt={includeCreatedAt}
  setIncludeCreatedAt={setIncludeCreatedAt}
/>
```

**Google Sheets 映射特点：**
- 问题标题自动作为第一行表头
- 每次同步自动更新表头（覆盖 A1 行）
- 数据追加模式（新回答添加到表格底部）
- 用户需确保对表格有编辑权限

---

## 五、字段同步策略

### 5.1 同步触发时机

数据同步通过 Pipeline 机制触发，在问卷提交后执行：

```typescript
// apps/web/app/api/(internal)/pipeline/lib/handleIntegrations.ts
export const handleIntegrations = async (
  integrations: TIntegration[],
  data: TPipelineInput,
  survey: TSurvey
) => {
  for (const integration of integrations) {
    switch (integration.type) {
      case "notion":
        await handleNotionIntegration(integration, data, survey);
        break;
      case "airtable":
        await handleAirtableIntegration(integration, data, survey);
        break;
      case "googleSheets":
        await handleGoogleSheetsIntegration(integration, data, survey);
        break;
      case "slack":
        await handleSlackIntegration(integration, data, survey);
        break;
    }
  }
};
```

---

### 5.2 通用数据处理流程

```typescript
const processDataForIntegration = async (
  integrationType,
  data,
  survey,
  includeVariables,
  includeMetadata,
  includeHiddenFields,
  includeCreatedAt,
  elementIds
) => {
  // 1. 提取问卷回答
  const { responses, elements } = await extractResponses(
    integrationType,
    data,
    elementIds,
    survey
  );

  // 2. 可选：附加元数据
  if (includeMetadata) {
    responses.push(convertMetaObjectToString(data.response.meta));
    elements.push("Metadata");
  }

  // 3. 可选：附加变量值
  if (includeVariables) {
    survey.variables.forEach((variable) => {
      const value = data.response.variables[variable.id];
      if (value !== undefined) {
        responses.push(String(value));
        elements.push(variable.name);
      }
    });
  }

  // 4. 可选：附加创建时间
  if (includeCreatedAt) {
    responses.push(getFormattedDateTimeString(new Date(data.response.createdAt)));
    elements.push("Created At");
  }

  return { responses, elements };
};

// 响应值提取
const extractResponses = async (integrationType, pipelineData, elementIds, survey) => {
  const responses = [];
  const elements = [];
  const surveyElements = getElementsFromBlocks(survey.blocks);

  for (const elementId of elementIds) {
    // 处理隐藏字段
    if (survey.hiddenFields.fieldIds?.includes(elementId)) {
      responses.push(processResponseData(pipelineData.response.data[elementId]));
      elements.push(elementId);
      continue;
    }

    const element = surveyElements.find((q) => q.id === elementId);
    if (!element) continue;

    const responseValue = pipelineData.response.data[elementId];

    // 根据问题类型处理
    if (element.type === TSurveyElementTypeEnum.PictureSelection) {
      const selectedChoiceIds = responseValue as string[];
      const urls = element.choices
        .filter((choice) => selectedChoiceIds.includes(choice.id))
        .map((choice) => resolveStorageUrlAuto(choice.imageUrl));
      responses.push(urls.join("\n"));
    } else if (element.type === TSurveyElementTypeEnum.FileUpload) {
      responses.push((responseValue as string[]).map(resolveStorageUrlAuto).join("; "));
    } else {
      responses.push(processResponseData(responseValue));
    }

    elements.push(getTextContent(element.headline));
  }

  return { responses, elements };
};
```

---

### 5.3 Notion 写入流程

```typescript
const handleNotionIntegration = async (integration, data, surveyData) => {
  for (const config of integration.config.data) {
    if (config.surveyId === data.surveyId) {
      // 1. 根据映射构建 payload
      const properties = buildNotionPayloadProperties(config.mapping, data, surveyData);

      // 2. 写入 Notion
      await writeNotionData(config.databaseId, properties, integration.config);
    }
  }
};

// Notion 类型适配
const getValue = (colType: string, value: any) => {
  switch (colType) {
    case "select":
      return { name: value?.replace(/,/g, "") };
    case "multi_select":
      return Array.isArray(value)
        ? value.map((v) => ({ name: v.replace(/,/g, "") }))
        : null;
    case "title":
      return [{ text: { content: value } }];
    case "rich_text":
      const content = Array.isArray(value) ? value.join("\n") : value;
      return [{ text: { content: truncateText(content, 2000) } }];
    case "checkbox":
      return value === "accepted" || value === "clicked";
    case "date":
      return { start: value };
    case "email":
      return value;
    case "number":
      return parseInt(value);
    case "phone_number":
      return value;
    case "url":
      return Array.isArray(value) ? value.join(", ") : value;
    default:
      return null;
  }
};

// 写入 API
export const writeData = async (databaseId, properties, config) => {
  await fetch(`https://api.notion.com/v1/pages`, {
    headers: getHeaders(config),
    method: "POST",
    body: JSON.stringify({
      parent: { database_id: databaseId },
      properties,
    }),
  });
};
```

**Notion 写入特点：**
- 创建新的 Database Page
- 丰富的类型适配（10+ 种类型）
- 不处理速率限制
- 失败不影响其他集成

---

### 5.4 Airtable 写入流程

```typescript
const handleAirtableIntegration = async (integration, data, survey) => {
  for (const config of integration.config.data) {
    if (config.surveyId === data.surveyId) {
      const values = await processDataForIntegration(
        "airtable",
        data,
        survey,
        config.includeVariables,
        config.includeMetadata,
        config.includeHiddenFields,
        config.includeCreatedAt,
        config.elementIds
      );
      await airtableWriteData(
        integration.config.key,
        config,
        values.responses,
        values.elements
      );
    }
  }
};

// Airtable 写入逻辑
export const writeData = async (key, configData, responses, elements) => {
  // 1. 构建记录数据
  const recordData: Record<string, string> = {};
  for (let i = 0; i < elements.length; i++) {
    recordData[elements[i]] =
      responses[i].length > AIRTABLE_MESSAGE_LIMIT
        ? truncateText(responses[i], AIRTABLE_MESSAGE_LIMIT)
        : responses[i];
  }

  // 2. 获取现有字段，确定需要创建的字段
  const existingFields = await getExistingFields(key, configData.baseId, configData.tableId);
  const fieldsToCreate = elements.filter((q) => !existingFields.has(q));

  // 3. 批量创建缺失字段（带速率限制控制）
  if (fieldsToCreate.length > 0) {
    const DELAY_BETWEEN_REQUESTS = 250; // 4 请求/秒，低于 Airtable 5 次/秒限制

    for (let i = 0; i < fieldsToCreate.length; i++) {
      const fieldName = fieldsToCreate[i];
      await addField(key, configData.baseId, configData.tableId, {
        name: fieldName,
        type: "singleLineText",
      });

      if (i < fieldsToCreate.length - 1) {
        await delay(DELAY_BETWEEN_REQUESTS);
      }
    }

    // 4. 等待字段创建完成（Airtable 最终一致性）
    await waitForFieldsToExist(key, configData, fieldsToCreate);
  }

  // 5. 写入记录
  await addRecords(key, configData.baseId, configData.tableId, recordData);
};

// 字段创建等待机制
async function waitForFieldsToExist(key, configData, fieldNames, maxRetries = 5, intervalMs = 2000) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const existingFields = await getExistingFields(key, configData.baseId, configData.tableId);
    const missingFields = fieldNames.filter((f) => !existingFields.has(f));

    if (missingFields.length === 0) return;

    if (attempt < maxRetries) {
      await new Promise((r) => setTimeout(r, intervalMs));
    }
  }
  throw new Error(`Timed out waiting for fields to exist`);
}

// 辅助函数
const addRecords = async (key, baseId, tableId, data) => {
  const req = await fetch(`https://api.airtable.com/v0/${baseId}/${tableId}`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key.access_token}`,
      "Content-type": "application/json",
    },
    body: JSON.stringify({ fields: data, typecast: true }),
  });
  return await req.json();
};

const addField = async (key, baseId, tableId, fieldConfig) => {
  const req = await fetch(`https://api.airtable.com/v0/meta/bases/${baseId}/tables/${tableId}/fields`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key.access_token}`,
      "Content-type": "application/json",
    },
    body: JSON.stringify(fieldConfig),
  });
  return await req.json();
};
```

**Airtable 写入特点：**
- 自动创建缺失字段（统一为 singleLineText 类型）
- 实现 API 速率限制控制（250ms 间隔 = 4 req/s）
- 处理 Airtable 最终一致性，最多 5 次重试等待字段创建完成
- 创建新 Record 记录

---

### 5.5 Google Sheets 写入流程

```typescript
const handleGoogleSheetsIntegration = async (integration, data, survey) => {
  for (const config of integration.config.data) {
    if (config.surveyId === data.surveyId) {
      const values = await processDataForIntegration(
        "googleSheets",
        data,
        survey,
        config.includeVariables,
        config.includeMetadata,
        config.includeHiddenFields,
        config.includeCreatedAt,
        config.elementIds
      );

      await writeData(
        integration,
        config.spreadsheetId,
        values.responses,
        values.elements
      );
    }
  }
};

// Google Sheets 写入逻辑
export const writeData = async (integrationData, spreadsheetId, responses, elements) => {
  const authClient = await authorize(integrationData);
  const sheets = google.sheets({ version: "v4", auth: authClient });

  // 截断过长文本
  const truncatedResponses = responses.map((response) =>
    response.length > GOOGLE_SHEET_MESSAGE_LIMIT
      ? truncateText(response, GOOGLE_SHEET_MESSAGE_LIMIT)
      : response
  );

  // 1. 更新表头（A1 行）
  sheets.spreadsheets.values.update(
    {
      spreadsheetId,
      range: "A1",
      valueInputOption: "RAW",
      resource: { values: [elements] },
    },
    (err) => {
      if (err) throw new Error(`Error updating headers: ${err.message}`);
    }
  );

  // 2. 追加数据（从 A2 开始）
  sheets.spreadsheets.values.append(
    {
      spreadsheetId,
      range: "A2",
      valueInputOption: "RAW",
      resource: { values: [truncatedResponses] },
    },
    (err) => {
      if (err) throw new Error(`Error appending data: ${err.message}`);
    }
  );
};
```

**Google Sheets 写入特点：**
- 自动更新表头（每次同步都覆盖 A1 行）
- 数据追加模式（新回答添加到表格底部）
- 使用 RAW 模式写入，不进行值类型推断
- 使用官方 Google API Client Library

---

## 六、三平台差异对照表

| 维度 | Notion | Airtable | Google Sheets |
|------|--------|----------|---------------|
| **鉴权方式** | OAuth 2.0 (Basic Auth) | OAuth 2.0 + PKCE | OAuth 2.0 (Google Client Library) |
| **Token 过期** | 永不过期 | ~60 分钟，自动刷新 | ~60 分钟，自动刷新 |
| **Token 刷新** | 无需 | refresh_token | refresh_token |
| **撤销检测** | 不检测 | 不检测 | 每次调用前 tokeninfo 验证 |
| **资源模型** | Database | Base → Table | Spreadsheet (URL 输入) |
| **资源发现** | Search API 搜索所有数据库 | Meta API 获取 Bases → Tables | 用户手动输入 URL |
| **字段映射方式** | 精确字段映射（选择目标字段） | 问题标题作为列名，自动创建 | 问题标题作为列名，自动更新表头 |
| **自动建列** | 否（需用户预先创建） | 是（singleLineText） | 是（自动更新 A1 行） |
| **同步触发** | 问卷提交后 Pipeline | 问卷提交后 Pipeline | 问卷提交后 Pipeline |
| **写入模式** | 创建新 Page | 创建新 Record | 追加新行 |
| **类型转换** | 丰富的类型适配（10+ 种） | 统一文本类型 | 统一文本类型 |
| **速率限制** | 未显式处理 | 4 请求/秒 控制（250ms 间隔） | 未显式处理 |
| **最终一致性** | 无需处理 | 需要（等待字段创建，最多 5 次重试） | 无需处理 |
| **权限检查点** | N/A | N/A | 用户输入 URL 后验证访问权限 |

---

## 七、架构设计总结

### 7.1 三层架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    前端 UI 层                            │
│  AddIntegrationModal - 资源选择与映射配置               │
│  MappingRow - 字段映射行组件（Notion 专用）              │
│  BaseSelectDropdown - Airtable Base 选择器               │
│  ElementCheckbox - 问题复选框（Airtable/Sheets）         │
│  AdditionalIntegrationSettings - 附加选项开关            │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                  API 接入层                              │
│  /api/v1/integrations/{type} - OAuth 授权发起           │
│  /api/v1/integrations/{type}/callback - OAuth 回调      │
│  /api/v1/integrations/airtable/tables - 获取 Table 列表 │
│  /api/(internal)/pipeline - 集成同步调度入口             │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                 服务层与数据层                           │
│  integration/service.ts - 集成 CRUD 操作                │
│  notion/service.ts - Notion API 封装                    │
│  airtable/service.ts - Airtable API 封装 + Token 管理   │
│  googleSheet/service.ts - Google Sheets API + Token 管理 │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                   Pipeline 执行层                        │
│  handleIntegrations.ts - 集成同步调度                   │
│  processDataForIntegration - 通用数据处理               │
│  extractResponses - 问题回答提取                         │
│  平台专用 writeData - 按平台特性写入                    │
└─────────────────────────────────────────────────────────┘
```

### 7.2 关键设计决策

| 决策点 | 方案选择 | 理由 |
|--------|----------|------|
| Token 存储 | Notion 加密 / 其他明文 | 历史遗留，Notion 先行实现 |
| 触发时机 | 问卷提交后异步执行 | 不影响用户提交体验 |
| 映射方式 | 问卷-资源绑定 + 平台差异化映射 | 适配各平台 API 能力差异 |
| 类型适配 | Notion 强类型 / 其他纯文本 | Notion API 要求严格类型 |
| 错误处理 | 单集成失败不影响其他集成 | 保证系统容错性 |
| Token 刷新 | 按需自动刷新 | 避免用户频繁重新授权 |
| 资源发现 | API 获取 vs 用户输入 | 适配各平台 API 能力差异 |

---

## 八、三平台落地检查清单

### 8.1 授权流程检查

| 检查项 | Notion | Airtable | Google Sheets | 验证方法 |
|--------|--------|----------|---------------|----------|
| ✅ 授权端点返回正确的 OAuth URL | ✅ | ✅ | ✅ | curl /api/v1/integrations/{type} |
| ✅ state 参数正确传递 environmentId | ✅ | ✅ | ✅ | 检查授权 URL |
| ✅ 回调接口正确换取 token | ✅ | ✅ | ✅ | 数据库验证 config.key 存在 |
| ✅ Notion token 已加密存储 | ✅ | - | - | 检查数据库 access_token 字段是否为密文 |
| ✅ Airtable PKCE 参数正确 | - | ✅ | - | code_challenge + code_challenge_method=S256 |
| ✅ Google Sheets offline/consent 参数 | - | - | ✅ | 检查授权 URL |

### 8.2 Token 生命周期管理

| 检查项 | Notion | Airtable | Google Sheets | 验证方法 |
|--------|--------|----------|---------------|----------|
| ✅ access_token 可正常调用 API | ✅ | ✅ | ✅ | 调用资源列表接口 |
| ✅ refresh_token 存储正确 | - | ✅ | ✅ | 检查数据库 config.key.refresh_token |
| ✅ expiry_date 正确计算 | - | ✅ | ✅ | 检查数据库 +60 分钟 |
| ✅ 过期自动刷新 | - | ✅ | ✅ | 修改 expiry_date 到过去，触发同步 |
| ✅ Google Sheets tokeninfo 验证 | - | - | ✅ | 撤销授权后触发同步 |
| ✅ 刷新后正确更新数据库 | - | ✅ | ✅ | 刷新前后对比 access_token |

### 8.3 资源选择与映射

| 检查项 | Notion | Airtable | Google Sheets | 验证方法 |
|--------|--------|----------|---------------|----------|
| ✅ 资源列表正常获取 | ✅ | ✅ | - | 前端集成页面 |
| ✅ Airtable Base/Table 级联选择 | - | ✅ | - | 前端操作验证 |
| ✅ Google Sheets URL 验证 | - | - | ✅ | 输入无效/无权限 URL |
| ✅ 问卷列表正确加载 | ✅ | ✅ | ✅ | 选择 environment 后验证 |
| ✅ Notion 字段双向映射 | ✅ | - | - | 选择数据库后查看字段下拉 |
| ✅ Airtable/Sheets 问题复选框 | - | ✅ | ✅ | 选择问卷后勾选问题 |
| ✅ 附加选项（变量/隐藏字段等） | ✅ | ✅ | ✅ | 勾选并验证同步结果 |
| ✅ 映射配置正确保存 | ✅ | ✅ | ✅ | 数据库检查 config.data |

### 8.4 写入前校验

| 检查项 | Notion | Airtable | Google Sheets | 验证方法 |
|--------|--------|----------|---------------|----------|
| ✅ surveyId 匹配检查 | ✅ | ✅ | ✅ | 提交其他问卷的回答 |
| ✅ elementIds 有效性检查 | ✅ | ✅ | ✅ | 删除问题后触发同步 |
| ✅ Notion 数据库存在 | ✅ | - | - | 删除数据库后触发同步 |
| ✅ Airtable Base/Table 存在 | - | ✅ | - | 删除 Base 后触发同步 |
| ✅ Google Sheets 编辑权限 | - | - | ✅ | 移除权限后触发同步 |
| ✅ 字段类型兼容性 | ✅ | - | - | 映射不兼容类型后触发 |
| ✅ 文本截断处理 | ✅ | ✅ | ✅ | 提交超长文本 |

### 8.5 失败处理

| 检查项 | Notion | Airtable | Google Sheets | 验证方法 |
|--------|--------|----------|---------------|----------|
| ✅ 单集成失败不影响其他集成 | ✅ | ✅ | ✅ | 一个集成失败时其他正常 |
| ✅ 错误日志记录 | ✅ | ✅ | ✅ | 查看服务端日志 |
| ✅ Airtable 字段创建重试 | - | ✅ | - | 新字段创建后立即写入 |
| ✅ Airtable 速率限制控制 | - | ✅ | - | 创建 10 个以上新字段 |
| ✅ Google Sheets invalid_grant 处理 | - | - | ✅ | 撤销授权后触发同步 |
| ✅ Pipeline 异常捕获 | ✅ | ✅ | ✅ | 故意制造异常不影响主流程 |
| ✅ 用户无感知失败 | ✅ | ✅ | ✅ | 集成失败不影响答卷提交 |
