# 环境与项目层级配置继承与覆盖关系分析

> **本文档结论均有直接代码依据，关键结论后附代码引用路径与行号**

---

## 一、术语演进与向后兼容

### 1.1 术语重命名映射

项目经历了从 `environment` / `project` 到 `workspace` 的术语重命名，但通过多层向后兼容机制保持对旧术语的支持：

| 新术语                | 旧术语（兼容）            | 所在模型 | 说明                          |
|----------------------|--------------------------|---------|-------------------------------|
| `workspaceId`        | `environmentId`          | Workspace | 工作空间/环境 ID            |
| `workspace`          | `environment` / `project`| -       | 工作空间配置对象              |
| `workspaceOverwrites`| `projectOverwrites`      | Survey  | 调查级行为覆盖配置            |
| `workspaceSettings`  | `projectSettings`        | -       | 工作空间设置                  |

### 1.2 ID 解析机制（两处解析，优先级不同）

#### 1.2.1 SDK 端参数解析（有明确优先级）

**代码依据**：`packages/js-core/src/lib/common/setup.ts:135`
```typescript
// Resolve effective ID: prefer workspaceId, fall back to environmentId
const effectiveId = configInput.workspaceId ?? configInput.environmentId;
```

**优先级（SDK 输入参数）**：
1. **`configInput.workspaceId`** - 新参数，最高优先级
2. **`configInput.environmentId`** - 旧参数，兜底兼容

**边界条件**：
- 同时提供两者时，`workspaceId` 完全覆盖 `environmentId`
  「代码依据」：`packages/js-core/src/lib/common/tests/setup.test.ts:172-204`
- 仅提供空字符串 `environmentId` 时，视为未提供
  「代码依据」：`packages/js-core/src/lib/common/tests/setup.test.ts:121-127`
- 两者都不提供时，返回 `missing_field` 错误
  「代码依据」：`packages/js-core/src/lib/common/setup.ts:137-143`

#### 1.2.2 服务端数据库查询（无严格先后顺序）

**代码依据**：`apps/web/lib/utils/resolve-client-id.ts:14-19`
```typescript
export const findWorkspaceByIdOrLegacyEnvId = async (id: string) => {
  return await prisma.workspace.findFirst({
    where: { OR: [{ id }, { legacyEnvironmentId: id }] },
    select: { id: true },
  });
};
```

**关键结论**：
- 使用 `OR` 条件**单次查询**，不是先查一个再查另一个
- `id` 和 `legacyEnvironmentId` 两个字段都有**唯一索引**，由 PostgreSQL 查询优化器决定使用哪个索引
- 若存在极端冲突（某个 ID 既是 A 的主键也是 B 的 legacyEnvironmentId），`findFirst` 返回哪条取决于数据库执行计划

---

### 1.3 向后兼容转换层

**代码依据**：`apps/web/app/lib/api/api-backwards-compat.ts`

#### 输入转换（请求体 → 内部格式）
| 输入情形                                  | 处理方式                                  |
|------------------------------------------|------------------------------------------|
| 仅 `projectOverwrites`                   | 映射为 `workspaceOverwrites`             |
| 同时 `projectOverwrites` + `workspaceOverwrites` | 丢弃 `projectOverwrites`，保留 `workspaceOverwrites` |
| 仅 `workspaceOverwrites`                 | 直接使用                                  |

#### 输出转换（内部格式 → 响应体）
- 响应中**同时包含** `workspace` 和 `project` 字段（值相同）
- 每个 survey **同时包含** `workspaceOverwrites` 和 `projectOverwrites` 字段（值相同）

「代码依据」：`apps/web/app/lib/api/api-backwards-compat.ts:60-65`

---

## 二、服务端配置加载流程

### 2.1 单次优化查询（关键性能设计）

**代码依据**：`apps/web/app/api/v1/client/[workspaceId]/environment/lib/data.ts:35-132`

`getWorkspaceStateData` 通过**单次 Prisma 查询**获取所有必要数据，避免 N+1 查询：

```typescript
const workspaceData = await prisma.workspace.findUnique({
  where: { id: workspaceId },
  select: {
    // Workspace 级字段
    id: true,
    recontactDays: true,        // Workspace 级
    clickOutsideClose: true,    // Workspace 级
    overlay: true,              // Workspace 级
    placement: true,            // Workspace 级
    inAppSurveyBranding: true,  // Workspace 级
    styling: true,              // Workspace 级

    // 关联查询：ActionClasses
    actionClasses: { select: { ... } },

    // 关联查询：Surveys（仅 app 类型、进行中）
    surveys: {
      where: { type: "app", status: "inProgress" },
      take: 30,  // 性能限制：最多 30 个调查
      select: {
        id: true,
        displayOption: true,     // ⚠️ Survey 级，Workspace 无此字段
        displayLimit: true,      // ⚠️ Survey 级，Workspace 无此字段
        recontactDays: true,     // Survey 级（覆盖 Workspace）
        workspaceOverwrites: true, // Survey 级行为覆盖
        styling: true,           // Survey 级样式
        // ... 其他字段
      },
    },
  },
});
```

### 2.2 字段归属真相（重要修正）

| 配置项               | 所在模型 | Workspace 有？ | Survey 有？ | 继承/覆盖关系 |
|---------------------|---------|---------------|------------|--------------|
| `recontactDays`     | 两者都有 | ✅ `Int @default(7)` | ✅ `Int?` | Survey 优先，null 时回退 Workspace |
| `displayOption`     | 仅 Survey | ❌ 无 | ✅ `displayOptions @default(displayOnce)` | 无继承，Survey 自有 |
| `displayLimit`      | 仅 Survey | ❌ 无 | ✅ `Int?` | 无继承，Survey 自有 |
| `placement`         | 仅 Workspace | ✅ | ⚠️ 需通过 `workspaceOverwrites` 覆盖 | Workspace 默认，Survey 可覆盖 |
| `overlay`           | 仅 Workspace | ✅ | ⚠️ 需通过 `workspaceOverwrites` 覆盖 | Workspace 默认，Survey 可覆盖 |
| `clickOutsideClose` | 仅 Workspace | ✅ | ⚠️ 需通过 `workspaceOverwrites` 覆盖 | Workspace 默认，Survey 可覆盖 |
| `inAppSurveyBranding` | 仅 Workspace | ✅ | ❌ 无 | 全局强制，不可覆盖 |

> **Schema 依据**：`packages/database/schema.prisma:371-373`（Survey 字段）、`:610`（Workspace 字段）
> **Type 依据**：`packages/types/js.ts:58-65`（WorkspaceSetting 无 displayOption/displayLimit）

---

### 2.3 多级缓存策略

**代码依据**：`apps/web/app/api/v1/client/[workspaceId]/environment/lib/environmentState.ts:24-71`

```typescript
export const getWorkspaceState = async (workspaceId: string) => {
  return cache.withCache(
    async () => { /* 查询逻辑 */ },
    createCacheKey.workspace.state(workspaceId),
    60 * 1000  // Redis TTL: 1 分钟
  );
};
```

**完整缓存层级（从快到慢）**：

| 层级 | 缓存位置 | TTL / 策略 | 代码依据 |
|------|---------|-----------|----------|
| 1 | SDK 内存（Config 单例） | 直到 `expiresAt`（1 小时） | `packages/js-core/src/lib/common/config.ts:6` |
| 2 | 浏览器 localStorage | 持久化，直到手动清除 | `packages/js-core/src/lib/common/config.ts:43-56` |
| 3 | 浏览器 HTTP Cache | `max-age=60`（1 分钟） | `apps/web/app/api/v1/client/[workspaceId]/environment/route.ts:77` |
| 4 | CDN（Cloudflare） | `s-maxage=60`（1 分钟） | 同上 |
| 5 | Redis（服务端） | 60 秒 | `environmentState.ts:70` |
| 6 | PostgreSQL | -（源数据） | - |

**附加缓存头**：
- `stale-while-revalidate=60`：过期后 60 秒内可返回陈旧数据同时后台刷新
- `stale-if-error=60`：源站出错时 60 秒内可返回陈旧数据

---

## 三、配置合并与优先级规则

### 3.1 配置层级全景

```
┌─────────────────────────────────────────────────────────────┐
│                    SDK 配置结构（TConfig）                   │
├─────────────────────────────────────────────────────────────┤
│  workspaceId: string                                         │
│  appUrl: string                                              │
│  status: { value: "success" | "error"; expiresAt: Date }     │
│                                                             │
│  ┌─ workspace (TWorkspaceState) ─┐                           │
│  │  expiresAt: Date              │  ← SDK 1 小时过期        │
│  │  data:                        │                           │
│  │    ├─ settings (Workspace 级) │                           │
│  │    │   ├─ recontactDays       │                           │
│  │    │   ├─ clickOutsideClose   │                           │
│  │    │   ├─ overlay             │                           │
│  │    │   ├─ placement           │                           │
│  │    │   ├─ inAppSurveyBranding │                           │
│  │    │   └─ styling             │                           │
│  │    │       └─ allowStyleOverwrite: boolean                │
│  │    │                          │                           │
│  │    ├─ surveys[] (Survey 级)   │                           │
│  │    │   ├─ displayOption       │  ← 仅 Survey 自有        │
│  │    │   ├─ displayLimit        │  ← 仅 Survey 自有        │
│  │    │   ├─ recontactDays       │  ← 可覆盖 Workspace      │
│  │    │   ├─ workspaceOverwrites │  ← 行为覆盖：placement等 │
│  │    │   └─ styling             │                           │
│  │    │       └─ overwriteThemeStyling?: boolean             │
│  │    │                          │                           │
│  │    └─ actionClasses[]         │                           │
│  └───────────────────────────────┘                           │
│                                                             │
│  ┌─ user (TUserState) ─┐                                     │
│  │  expiresAt: Date   │                                     │
│  │  data: { userId, contactId, segments, displays, ... }   │
│  └─────────────────────┘                                     │
│                                                             │
│  filteredSurveys: TWorkspaceStateSurvey[]  ← 运行时过滤结果 │
└─────────────────────────────────────────────────────────────┘
```

---

### 3.2 行为配置优先级（placement / overlay / clickOutsideClose）

**代码依据**：`packages/js-core/src/lib/survey/widget.ts:105-108`

```typescript
const workspaceOverwrites = survey.workspaceOverwrites ?? {};
const clickOutside = workspaceOverwrites.clickOutsideClose ?? settings.clickOutsideClose;
const overlay = workspaceOverwrites.overlay ?? settings.overlay;
const placement = workspaceOverwrites.placement ?? settings.placement;
```

**优先级（从高到低）**：
1. `survey.workspaceOverwrites.placement` - 调查级覆盖（若不为 null/undefined）
2. `settings.placement` - 工作空间全局默认

**空值判定逻辑**：
- 使用 `??`（nullish coalescing），仅 `null` / `undefined` 时回退
- `false`、`0`、`""` 等 falsy 值**不会**触发回退

---

### 3.3 样式配置优先级（双重开关机制）

**代码依据**：`packages/js-core/src/lib/common/utils.ts:150-167`

```typescript
export const getStyling = (
  settings: TWorkspaceStateSettings,
  survey: TWorkspaceStateSurvey
): TWorkspaceStyling | TSurveyStyling => {
  if (settings.styling.allowStyleOverwrite) {
    if (!survey.styling?.overwriteThemeStyling) {
      return settings.styling;  // Workspace 允许覆盖，但 Survey 未开启
    }
    return survey.styling;     // 两者都开启，使用 Survey 样式
  }
  return settings.styling;     // Workspace 不允许覆盖
};
```

**真值表**：

| `workspace.styling.allowStyleOverwrite` | `survey.styling.overwriteThemeStyling` | 最终样式来源 |
|----------------------------------------|----------------------------------------|-------------|
| `false`                                | 任意值（true/false/null/undefined）    | Workspace   |
| `true`                                 | `false` / `null` / `undefined`        | Workspace   |
| `true`                                 | `true`                                 | Survey      |

**设计意图**：
- `allowStyleOverwrite` 是**管理员级总开关**，可完全禁止调查级样式自定义
- `overwriteThemeStyling` 是**调查级开关**，即使全局允许，单个调查也可选择不覆盖

---

### 3.4 recontactDays 优先级（两级继承）

**代码依据**：`packages/js-core/src/lib/common/utils.ts:109-131`

```typescript
filteredSurveys = filteredSurveys.filter((survey) => {
  if (!lastDisplayAt) return true;  // 从未显示过，不限制

  // Survey 级 recontactDays 优先
  if (survey.recontactDays !== null) {
    return diffInDays(new Date(), new Date(lastDisplayAt)) >= survey.recontactDays;
  }

  // 回退到 Workspace 级
  if (settings.recontactDays) {
    return diffInDays(new Date(), new Date(lastDisplayAt)) >= settings.recontactDays;
  }

  return true;  // 都未设置，不限制
});
```

**优先级**：
1. `survey.recontactDays`（非 null 时）
2. `settings.recontactDays`（非 0 时）
3. 无限制

---

### 3.5 displayOption 与 displayLimit（Survey 自有，无继承）

**代码依据**：`packages/js-core/src/lib/common/utils.ts:81-106`

```typescript
let filteredSurveys = surveys.filter((survey: TWorkspaceStateSurvey) => {
  switch (survey.displayOption) {  // ⚠️ 直接读 survey 字段，无回退
    case "respondMultiple":
      return true;
    case "displayOnce":
      return displays.filter(d => d.surveyId === survey.id).length === 0;
    case "displayMultiple":
      return responses.filter(id => id === survey.id).length === 0;
    case "displaySome":
      if (survey.displayLimit === null) return true;
      if (responses.filter(id => id === survey.id).length) return false;
      return displays.filter(d => d.surveyId === survey.id).length < survey.displayLimit;
    default:
      throw Error("Invalid displayOption");
  }
});
```

**关键结论**：
- `displayOption` 和 `displayLimit` **仅存在于 Survey 模型**，Workspace 没有对应字段
- 没有任何回退或继承逻辑，每个 Survey 独立配置
- `displayOption` 有默认值 `displayOnce`（Prisma schema 中定义）
- `displayLimit` 可为 `null`，此时 `displaySome` 等同于 `displayMultiple`

---

## 四、运行时参数影响范围

### 4.1 运行时参数定义

**代码依据**：`packages/js-core/src/types/survey.ts:92-94`

```typescript
export interface TTrackProperties {
  hiddenFields: Record<string, string | number | string[]>;
}
```

**运行时参数只有 `hiddenFields` 一个字段**，且代码中多次标记为 `deprecated`：

> `@param properties - Optional properties to set, like the hidden fields (deprecated, hidden fields will be removed in a future version)`
> 「代码依据」：`packages/js-core/src/lib/survey/action.ts:11`、`:47`、`packages/js-core/src/index.ts:76`

---

### 4.2 hiddenFields 处理逻辑

**代码依据**：`packages/js-core/src/lib/common/utils.ts:272-302`

```typescript
export const handleHiddenFields = (
  hiddenFieldsConfig: TWorkspaceStateSurvey["hiddenFields"],
  hiddenFields?: TTrackProperties["hiddenFields"]
): TTrackProperties["hiddenFields"] => {
  const { enabled: enabledHiddenFields, fieldIds: surveyHiddenFieldIds } = hiddenFieldsConfig;

  let hiddenFieldsObject = {};

  if (!enabledHiddenFields) {
    logger.error("Hidden fields are not enabled for this survey");
  } else if (surveyHiddenFieldIds && hiddenFields) {
    // 只保留在 survey.fieldIds 白名单中的字段
    hiddenFieldsObject = Object.keys(hiddenFields).reduce((acc, key) => {
      if (surveyHiddenFieldIds.includes(key)) {
        acc[key] = hiddenFields[key];
      } else {
        // 不在白名单中的字段打 error 日志并丢弃
        unknownHiddenFields.push(key);
      }
      return acc;
    }, {});
  }

  return hiddenFieldsObject;
};
```

**运行时参数影响边界**：
| 可影响 | 不可影响 |
|--------|----------|
| ✅ hiddenFields 白名单内的字段值 | ❌ 任何样式配置 |
| | ❌ 任何行为配置（placement、overlay等） |
| | ❌ displayOption、displayLimit |
| | ❌ recontactDays |
| | ❌ 调查过滤逻辑 |
| | ❌ 工作空间级任何设置 |

---

## 五、客户端 SDK 配置管理

### 5.1 Config 单例与合并策略

**代码依据**：`packages/js-core/src/lib/common/config.ts:23-34`

```typescript
public update(newConfig: TConfigUpdateInput): void {
  this.config = {
    ...this.config,    // 旧配置
    ...newConfig,      // 新配置覆盖
    status: {          // status 特殊合并逻辑
      value: newConfig.status?.value ?? "success",
      expiresAt: newConfig.status?.expiresAt ?? null,
    },
  };
  void this.saveToStorage();
}
```

**合并规则**：
1. **浅合并**：`{ ...existing, ...new }` - 顶层字段新值完全覆盖旧值
2. **深层对象无递归合并**：如果 `newConfig.workspace` 只传了部分字段，**未传字段会丢失**（被 undefined 覆盖）
3. **status 特殊处理**：
   - `status.value`：新值优先，默认 `"success"`
   - `status.expiresAt`：新值优先，默认 `null`

> **重要边界**：`update()` 方法是**非增量更新**。调用方必须保证传入完整的配置对象，否则未传的顶层字段会变成 `undefined`。

---

### 5.2 初始化流程（setup）

**代码依据**：`packages/js-core/src/lib/common/setup.ts:72-342`

```
1. LocalStorage 迁移检查
   ├─ 旧字段 environmentId → workspaceId
   ├─ 旧字段 environment → workspace
   └─ 旧字段 project → settings

2. 已初始化检查：getIsSetup() === true → 直接返回

3. 错误状态检查
   ├─ status.value === "error" 且 isDebug === false
   │   ├─ status.expiresAt 未过期 → 跳过初始化
   │   └─ status.expiresAt 已过期 → 继续
   └─ isDebug === true → 忽略错误状态，重置配置

4. 现有配置匹配检查
   ├─ existingConfig.workspaceId === effectiveId
   │  && existingConfig.appUrl === configInput.appUrl
   │  && existingConfig.status.value !== "error"
   │   → 检查过期时间：
   │     ├─ workspace.expiresAt 过期 → fetchWorkspaceState
   │     └─ user.expiresAt 过期 → fetch 用户状态
   └─ 不匹配 → 完整初始化：
        ├─ fetchWorkspaceState（服务端）
        ├─ 默认 userState（无 userId）
        └─ 有 userId 时 sendUpdatesToBackend

5. filterSurveys(workspace, userState)

6. config.update({ workspace, user, filteredSurveys })

7. setIsSetup(true)
```

---

### 5.3 错误状态机制

**代码依据**：`packages/js-core/src/lib/common/setup.ts:363-406`

```typescript
export const handleErrorOnFirstSetup = (e) => {
  // 首次 setup 失败时，写入错误状态到 localStorage
  const initialErrorConfig = {
    status: {
      value: "error",
      expiresAt: new Date(Date.now() + 10 * 60000),  // 10 分钟
    },
  };
  localStorage.setItem(JS_LOCAL_STORAGE_KEY, JSON.stringify(initialErrorConfig));
  throw new Error("Could not set up formbricks");
};

export const putFormbricksInErrorState = (formbricksConfig) => {
  // 运行中出错时，更新配置为错误状态
  formbricksConfig.update({
    ...formbricksConfig.get(),
    status: { value: "error", expiresAt: new Date(Date.now() + 10 * 60000) },
  });
  tearDown();
};
```

**错误状态边界条件**：
- 错误状态有效期：**10 分钟**
- Debug 模式（`?fb_debug=true`）下**忽略错误状态**
- 错误状态下 SDK 完全停止工作，不显示任何调查
- 过期后自动恢复重试

---

### 5.4 命令队列与执行顺序

**代码依据**：`packages/js-core/src/lib/common/command-queue.ts`

所有 SDK 公开方法（`setup`、`setUserId`、`track` 等）都通过命令队列序列化执行：

```typescript
// packages/js-core/src/index.ts:14
const setup = async (setupConfig) => {
  await queue.add(Setup.setup, CommandType.Setup, false, setupConfig);
  await queue.wait();
};

const track = async (code, properties) => {
  await queue.add(Action.trackCodeAction, CommandType.GeneralAction, true, code, properties);
};
```

**队列规则**：
1. **FIFO 顺序执行**：保证 `setup()` 完成后才执行后续命令
2. **UserAction 类型**：需检查 setup 完成标记，未完成则跳过
3. **GeneralAction 类型**：执行前会等待 `UpdateQueue`（用户识别队列）完成
4. **错误静默**：命令失败仅打 error 日志，不抛出异常

---

### 5.5 字段重映射（服务端 → 客户端）

**代码依据**：`packages/js-core/src/lib/workspace/state.ts:43-51`

服务端响应使用 `data.workspace`，但客户端内部使用 `data.settings` 避免 `workspace.workspace` 嵌套：

```typescript
const rawData = response.data as TWorkspaceState & {
  data: { workspace?: TWorkspaceState["data"]["settings"] };
};

if (rawData.data.workspace && !rawData.data.settings) {
  rawData.data.settings = rawData.data.workspace;
  delete rawData.data.workspace;
}
```

---

## 六、关键边界条件汇总

| 边界场景 | 行为 | 代码依据 |
|---------|------|----------|
| `workspaceId` 与 `environmentId` 同时提供 | `workspaceId` 完全覆盖 `environmentId` | `setup.ts:135`、`setup.test.ts:172-204` |
| `legacyEnvironmentId` 与其他 `id` 冲突 | `findFirst` 由数据库执行计划决定返回 | `resolve-client-id.ts:14-19` |
| `workspaceOverwrites.placement = false` | 不会回退到 Workspace（`??` 只判 null/undefined） | `widget.ts:108` |
| `survey.recontactDays = 0` | 优先使用 0（不限制），不会回退到 Workspace | `utils.ts:119-121` |
| `survey.displayLimit = null` | `displaySome` 等同于 `displayMultiple`（无次数限制） | `utils.ts:92-94` |
| `allowStyleOverwrite = false` | `overwriteThemeStyling = true` 也无效，强制使用 Workspace 样式 | `utils.ts:155-166` |
| 运行时传入未在白名单的 hiddenField | 打 error 日志并默默丢弃 | `utils.ts:284-298` |
| 首次 setup 网络失败 | 进入 10 分钟错误状态，期间不重试 | `setup.ts:363-385` |
| Debug 模式激活 | 忽略错误状态，跳过过期检查 | `setup.ts:113-119` |
| `displaySome` 且已有 response | 不再显示，即使 displayLimit 未达 | `utils.ts:97-99` |

---

## 七、优先级总览（修正版）

### 7.1 行为配置优先级

```
┌─────────────────────────────────────────────┐
│  最高：SDK 调用时的 runtime 参数（仅 hiddenFields）│
├─────────────────────────────────────────────┤
│  第二：Survey.workspaceOverwrites             │
│        （placement / overlay / clickOutsideClose）│
├─────────────────────────────────────────────┤
│  第三：Workspace.settings                     │
│        （placement / overlay / clickOutsideClose）│
└─────────────────────────────────────────────┘
```

### 7.2 样式配置优先级（双重开关）

```
┌─────────────────────────────────────────────┐
│  Workspace.styling.allowStyleOverwrite       │
│        ├─ false → 强制 Workspace 样式        │
│        └─ true                               │
│              ├─ Survey.styling.overwriteThemeStyling │
│              │     ├─ !== true → Workspace 样式 │
│              │     └─ === true → Survey 样式    │
└─────────────────────────────────────────────┘
```

### 7.3 displayOption / displayLimit 真值表

| displayOption | 显示条件 |
|--------------|---------|
| `respondMultiple` | 总是显示（只要未被 segment 过滤） |
| `displayOnce` | 该 survey 的 displays 计数 === 0 |
| `displayMultiple` | 该 survey 不在 responses 列表中 |
| `displaySome` | 不在 responses 中 且 displays 计数 < displayLimit |

---

## 八、关键代码位置速查表

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| SDK 端 ID 优先级（workspaceId > environmentId） | `packages/js-core/src/lib/common/setup.ts` | 135 |
| 服务端 ID OR 查询（无严格先后） | `apps/web/lib/utils/resolve-client-id.ts` | 14-19 |
| 向后兼容输入转换 | `apps/web/app/lib/api/api-backwards-compat.ts` | 17-30 |
| 向后兼容输出转换 | `apps/web/app/lib/api/api-backwards-compat.ts` | 39-65 |
| 单次数据库查询 | `apps/web/app/api/v1/client/[workspaceId]/environment/lib/data.ts` | 35-132 |
| Workspace 级字段（无 displayOption/displayLimit） | `packages/types/js.ts` | 58-65 |
| Survey 级字段（含 displayOption/displayLimit） | `packages/types/js.ts` | 9-34 |
| 行为配置优先级 | `packages/js-core/src/lib/survey/widget.ts` | 105-108 |
| 样式配置优先级（双重开关） | `packages/js-core/src/lib/common/utils.ts` | 150-167 |
| recontactDays 优先级 | `packages/js-core/src/lib/common/utils.ts` | 109-131 |
| displayOption 处理逻辑 | `packages/js-core/src/lib/common/utils.ts` | 81-106 |
| 运行时 hiddenFields 白名单过滤 | `packages/js-core/src/lib/common/utils.ts` | 272-302 |
| Config 合并策略（浅合并+status 特殊处理） | `packages/js-core/src/lib/common/config.ts` | 23-34 |
| 错误状态处理 | `packages/js-core/src/lib/common/setup.ts` | 363-406 |
| 命令队列执行 | `packages/js-core/src/lib/common/command-queue.ts` | 74-118 |
| 服务端 → 客户端字段重映射 | `packages/js-core/src/lib/workspace/state.ts` | 43-51 |
| Prisma Schema（Survey 字段） | `packages/database/schema.prisma` | 371-373 |
| Prisma Schema（Workspace 字段） | `packages/database/schema.prisma` | 610 |

---

## 九、常见误解澄清

| 误解 | 真相 | 代码依据 |
|-----|------|----------|
| "displayOption 有 Workspace 级默认值" | ❌ displayOption 仅存在于 Survey，有 schema 默认值 `displayOnce` | `schema.prisma:371`、`js.ts:58-65` |
| "ID 解析先查 workspace.id 再查 legacyEnvironmentId" | ❌ 单次 OR 查询，数据库优化器决定，无固定先后 | `resolve-client-id.ts:14-19` |
| "运行时 properties 可覆盖样式" | ❌ 仅能传 hiddenFields，且已标记 deprecated | `types/survey.ts:92-94`、`index.ts:76` |
| "config.update() 是增量更新" | ❌ 浅合并，未传的顶层字段会丢失 | `config.ts:23-34` |
| "Workspace 级有 displayLimit" | ❌ 仅 Survey 有 | `js.ts:58-65` |
| "`??` 会把 false 当作空值回退" | ❌ `??` 只对 `null`/`undefined` 回退，`false` 是有效值 | `widget.ts:105-108` |
