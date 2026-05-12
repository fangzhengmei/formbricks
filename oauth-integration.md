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
access_token 加密后存储到数据库
    ↓
重定向回集成配置页面
```

### 1.2 核心实现细节

#### 授权端点 (`apps/web/app/api/v1/integrations/notion/route.ts`)

```typescript
// 获取授权URL
export const GET = withV1ApiWrapper({
  handler: async ({ req, authentication }) => {
    const environmentId = req.headers.get("environmentId");
    
    // 构建 OAuth 授权 URL
    const authUrl = `${NOTION_AUTH_URL}&state=${environmentId}`;
    
    return { response: responses.successResponse({ authUrl }) };
  }
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
    const tokenResponse = await fetch("https://api.notion.com/v1/oauth/token", {
      method: "POST",
      headers: { Authorization: `Basic ${encodedCredentials}` },
      body: JSON.stringify({
        grant_type: "authorization_code",
        code,
        redirect_uri: NOTION_REDIRECT_URI
      })
    });
    
    const tokenData = await tokenResponse.json();
    
    // 2. 加密存储敏感凭证
    const encryptedAccessToken = symmetricEncrypt(tokenData.access_token, ENCRYPTION_KEY);
    tokenData.access_token = encryptedAccessToken;
    
    // 3. 存储到数据库
    await createOrUpdateIntegration(environmentId, {
      type: "notion",
      config: {
        key: tokenData,
        data: []
      }
    });
    
    // 4. 重定向回集成页面
    return { response: Response.redirect(`${WEBAPP_URL}/environments/${environmentId}/workspace/integrations/notion`) };
  }
});
```

### 1.3 加密策略

- 使用 `symmetricEncrypt` 函数对 access_token 进行 AES 加密
- 加密密钥存储在环境变量 `ENCRYPTION_KEY`
- 数据库中只存储加密后的令牌，避免明文泄露

---

## 二、资源选择与映射

### 2.1 数据模型设计

#### 集成基础类型 (`packages/types/integration/shared-types.ts`)

```typescript
// 基础集成结构
interface ZIntegrationBase {
  id: string;
  environmentId: string;
}

// 问卷映射基础数据
interface ZIntegrationBaseSurveyData {
  createdAt: Date;
  elementIds: string[];           // 要同步的问卷元素ID
  includeVariables: boolean;      // 是否包含变量
  includeHiddenFields: boolean;   // 是否包含隐藏字段
  includeMetadata: boolean;       // 是否包含元数据
  includeCreatedAt: boolean;      // 是否包含创建时间
  elements: string;               // 元素名称
  surveyId: string;               // 关联问卷ID
  surveyName: string;             // 问卷名称
}
```

#### Notion 特定配置 (`packages/types/integration/notion.ts`)

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

### 2.2 资源选择流程

#### 步骤1：获取第三方资源列表

以 Notion 为例，调用 Notion Search API 获取用户有权限访问的数据库：

```typescript
// apps/web/lib/notion/service.ts
export const getNotionDatabases = async (environmentId: string) => {
  const notionIntegration = await getIntegrationByType(environmentId, "notion");
  
  const res = await fetch("https://api.notion.com/v1/search", {
    headers: getHeaders(notionIntegration.config),
    method: "POST",
    body: JSON.stringify({
      page_size: 100,
      filter: { value: "database", property: "object" }
    })
  });
  
  return (await res.json()).results;
};
```

#### 步骤2：前端资源选择界面

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

// 3. 获取数据库字段供映射使用
const dbItems = Object.keys(selectedDatabase.properties).map(fieldKey => ({
  id: selectedDatabase.properties[fieldKey].id,
  name: selectedDatabase.properties[fieldKey].name,
  type: selectedDatabase.properties[fieldKey].type
}));

// 4. 获取问卷元素供映射使用
const elementItems = elements.map(el => ({
  id: el.id,
  name: getTextContent(el.headline),
  type: el.type
}));
```

### 2.3 字段映射界面

```typescript
// MappingRow 组件实现双向映射
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

映射验证规则：
- 至少需要配置一组映射
- 每个映射必须同时选择 Formbricks 字段和 Notion 字段
- 不允许重复映射同一个字段
- 类型兼容性检查（如日期字段只能映射到日期类型）

---

## 三、字段同步策略

### 3.1 同步触发时机

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

### 3.2 数据处理流程

```typescript
const processDataForIntegration = async (
  integrationType: TIntegrationType,
  data: TPipelineInput,
  survey: TSurvey,
  includeVariables: boolean,
  includeMetadata: boolean,
  includeHiddenFields: boolean,
  includeCreatedAt: boolean,
  elementIds: string[]
) => {
  // 1. 提取问卷回答
  const { responses, elements } = await extractResponses(integrationType, data, elementIds, survey);
  
  // 2. 可选：附加元数据
  if (includeMetadata) {
    responses.push(convertMetaObjectToString(data.response.meta));
    elements.push("Metadata");
  }
  
  // 3. 可选：附加变量值
  if (includeVariables) {
    survey.variables?.forEach(variable => {
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
```

### 3.3 响应值提取逻辑

```typescript
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
    
    const element = surveyElements.find(q => q.id === elementId);
    if (!element) continue;
    
    const responseValue = pipelineData.response.data[elementId];
    
    // 根据问题类型处理不同的响应值
    if (element.type === TSurveyElementTypeEnum.PictureSelection) {
      const selectedChoiceIds = responseValue as string[];
      const urls = element.choices
        .filter(choice => selectedChoiceIds.includes(choice.id))
        .map(choice => resolveStorageUrlAuto(choice.imageUrl));
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

### 3.4 Notion 类型适配

Notion API 对不同字段类型有特定的 payload 格式要求：

```typescript
const getValue = (colType: string, value: any) => {
  switch (colType) {
    case "select":
      return { name: value?.replace(/,/g, "") };
      
    case "multi_select":
      return Array.isArray(value) 
        ? value.map(v => ({ name: v.replace(/,/g, "") })) 
        : null;
        
    case "title":
      return [{ text: { content: value } }];
      
    case "rich_text":
      const content = Array.isArray(value) ? value.join("\n") : value;
      return [{ 
        text: { 
          content: truncateText(content, NOTION_RICH_TEXT_LIMIT) 
        } 
      }];
      
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
```

### 3.5 同步执行

```typescript
const handleNotionIntegration = async (integration, data, surveyData) => {
  for (const element of integration.config.data) {
    // 只处理匹配的问卷
    if (element.surveyId === data.surveyId) {
      // 1. 根据映射构建 payload
      const properties = buildNotionPayloadProperties(element.mapping, data, surveyData);
      
      // 2. 写入 Notion
      await writeNotionData(element.databaseId, properties, integration.config);
    }
  }
};

// 写入 Notion API
export const writeData = async (databaseId, properties, config) => {
  await fetch("https://api.notion.com/v1/pages", {
    headers: getHeaders(config),
    method: "POST",
    body: JSON.stringify({
      parent: { database_id: databaseId },
      properties
    })
  });
};
```

---

## 四、架构设计总结

### 4.1 三层架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    前端 UI 层                            │
│  AddIntegrationModal - 资源选择与映射配置               │
│  MappingRow - 字段映射行组件                            │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                  API 接入层                              │
│  /api/v1/integrations/{type} - OAuth 授权发起           │
│  /api/v1/integrations/{type}/callback - OAuth 回调      │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                 服务层与数据层                           │
│  integration/service.ts - 集成CRUD操作                  │
│  notion/service.ts - Notion API 封装                    │
│  airtable/service.ts - Airtable API 封装                │
│  googleSheet/service.ts - Google Sheets API 封装        │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│                   Pipeline 执行层                        │
│  handleIntegrations.ts - 集成同步调度                   │
│  按映射规则转换数据 → 调用第三方API写入                 │
└─────────────────────────────────────────────────────────┘
```

### 4.2 关键设计决策

| 决策点 | 方案选择 | 理由 |
|--------|----------|------|
| Token 存储 | 加密后存储数据库 | 符合安全规范，避免明文泄露 |
| 触发时机 | 问卷提交后异步执行 | 不影响用户提交体验 |
| 映射方式 | 问卷-资源双向绑定 + 字段映射 | 灵活支持多问卷同步到不同资源 |
| 类型适配 | 平台专用转换函数 | 处理各平台 API 的格式差异 |
| 错误处理 | 单集成失败不影响其他集成 | 保证系统容错性 |

### 4.3 支持的集成类型

| 平台 | 授权方式 | 目标资源 | 特色 |
|------|----------|----------|------|
| Notion | OAuth 2.0 | Database | 丰富的字段类型支持 |
| Airtable | OAuth 2.0 | Base → Table | 工作空间级授权 |
| Google Sheets | OAuth 2.0 | Spreadsheet | 电子表格原生体验 |
| Slack | OAuth 2.0 | Channel | 实时消息通知 |

---

## 五、开发扩展指南

### 5.1 新增集成类型步骤

1. **定义类型**：在 `packages/types/integration/` 下创建新集成的类型定义
2. **实现 OAuth 流程**：创建授权端点和回调端点
3. **实现资源 API**：封装获取目标资源列表的 API
4. **配置 UI**：创建 AddIntegrationModal 组件
5. **同步逻辑**：在 `handleIntegrations.ts` 中添加处理函数
6. **数据写入**：实现平台专用的 writeData 函数

### 5.2 注意事项

- 所有 access_token 必须加密存储
- 实现 token 刷新机制（如 Airtable 和 Google Sheets）
- 添加完善的错误日志
- 考虑 API 速率限制
- 支持用户断开集成时清理数据
