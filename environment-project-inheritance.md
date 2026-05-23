# 环境与项目层级配置继承与覆盖关系分析

## 一、术语演进与向后兼容

### 1.1 术语重命名背景

项目经历了从 `environment` / `project` 到 `workspace` 的术语重命名，但通过向后兼容层保持了对旧术语的支持：

| 新术语       | 旧术语（兼容）       | 说明                     |
|-------------|---------------------|--------------------------|
| workspaceId | environmentId       | 工作空间/环境 ID         |
| workspace   | environment / project | 工作空间配置对象       |
| workspaceOverwrites | projectOverwrites | 调查级覆盖配置       |
| workspaceSettings | projectSettings | 工作空间设置         |

### 1.2 ID 解析机制

**文件**：`apps/web/lib/utils/resolve-client-id.ts:28`

```typescript
export const resolveClientApiIds = reactCache(async (id: string): Promise<TResolvedClientIds | null> => {
  const workspace = await findWorkspaceByIdOrLegacyEnvId(id);
  if (workspace) {
    return { workspaceId: workspace.id };
  }
  return null;
});
```

**优先级**：
1. 首先尝试按 `workspace.id` 主键查找
2. 失败后回退到 `workspace.legacyEnvironmentId` 查找
3. 两个字段都有唯一索引，查询效率相同

### 1.3 向后兼容层

**文件**：`apps/web/app/lib/api/api-backwards-compat.ts`

**输入转换（请求体）**：
- 仅提供 `projectOverwrites` → 映射为 `workspaceOverwrites`
- 同时提供两者 → 丢弃 `projectOverwrites`，保留 `workspaceOverwrites`
- 仅提供 `workspaceOverwrites` → 直接使用

**输出转换（响应体）**：
- 响应中同时包含 `workspace` 和 `project`（值相同）
- 每个 survey 同时包含 `workspaceOverwrites` 和 `projectOverwrites`（值相同）

---

## 二、服务端配置加载流程

### 2.1 单次优化查询

**文件**：`apps/web/app/api/v1/client/[workspaceId]/environment/lib/data.ts:35`

`getWorkspaceStateData` 函数通过单次 Prisma 查询获取所有必要数据，避免多次数据库往返：

```typescript
const workspaceData = await prisma.workspace.findUnique({
  where: { id: workspaceId },
  select: {
    id: true,
    appSetupCompleted: true,
    recontactDays: true,
    clickOutsideClose: true,
    overlay: true,
    placement: true,
    inAppSurveyBranding: true,
    styling: true,
    actionClasses: { select: { ... } },
    surveys: {
      where: { type: "app", status: "inProgress" },
      orderBy: { createdAt: "desc" },
      take: 30,
      select: {
        id: true,
        workspaceOverwrites: true,  // 调查级覆盖
        styling: true,               // 调查级样式
        // ... 其他字段
      },
    },
  },
});
```

### 2.2 缓存机制

**文件**：`apps/web/app/api/v1/client/[workspaceId]/environment/lib/environmentState.ts:24`

```typescript
export const getWorkspaceState = async (workspaceId: string) => {
  return cache.withCache(
    async () => {
      const { workspace, surveys, actionClasses } = await getWorkspaceStateData(workspaceId);
      // ... 数据处理
      const data = addLegacyProjectToEnvironmentState({
        surveys: addLegacyProjectOverwritesToList(surveys),
        actionClasses,
        workspace: workspace.workspaceSettings,
      });
      return { data };
    },
    createCacheKey.workspace.state(workspaceId),
    60 * 1000  // Redis 缓存 1 分钟
  );
};
```

**多级缓存策略**：
1. **Redis 缓存**（服务端）：1 分钟 TTL
2. **CDN 缓存**（Cloudflare）：`s-maxage=60`
3. **浏览器缓存**：`max-age=60`
4. **stale-while-revalidate**：60 秒
5. **stale-if-error**：60 秒

### 2.3 API 响应结构

**文件**：`apps/web/app/api/v1/client/[workspaceId]/environment/route.ts:66`

```typescript
return {
  response: responses.successResponse(
    {
      data,                // workspace 数据（含 surveys, actionClasses, settings）
      expiresAt: new Date(Date.now() + 1000 * 60 * 60),  // SDK 1 小时后重新检查
    },
    true,
    "public, s-maxage=60, max-age=60, stale-while-revalidate=60, stale-if-error=60"
  ),
};
```

---

## 三、配置合并与优先级规则

### 3.1 配置层级结构

```
Workspace（工作空间）
├── settings（全局设置）
│   ├── recontactDays
│   ├── clickOutsideClose
│   ├── overlay
│   ├── placement
│   ├── inAppSurveyBranding
│   └── styling（全局样式）
│       ├── allowStyleOverwrite: boolean  ← 样式覆盖总开关
│       └── ... 其他样式字段
│
└── surveys[]（调查列表）
    ├── workspaceOverwrites  ← 行为配置覆盖
    │   ├── clickOutsideClose
    │   ├── overlay
    │   └── placement
    └── styling  ← 样式配置
        ├── overwriteThemeStyling: boolean | null  ← 单调查样式覆盖开关
        └── ... 其他样式字段
```

### 3.2 行为配置优先级（placement, overlay, clickOutsideClose）

**文件**：`packages/js-core/src/lib/survey/widget.ts:105-108`

```typescript
const workspaceOverwrites = survey.workspaceOverwrites ?? {};
const clickOutside = workspaceOverwrites.clickOutsideClose ?? settings.clickOutsideClose;
const overlay = workspaceOverwrites.overlay ?? settings.overlay;
const placement = workspaceOverwrites.placement ?? settings.placement;
```

**优先级**（从高到低）：
1. **survey.workspaceOverwrites.clickOutsideClose** - 调查级覆盖
2. **settings.clickOutsideClose** - 工作空间全局设置

**判定逻辑**：
- 如果 `survey.workspaceOverwrites` 存在且字段不为 `null` / `undefined`，使用调查级值
- 否则回退到工作空间全局设置

### 3.3 样式配置优先级（双重开关机制）

**文件**：`packages/js-core/src/lib/common/utils.ts:150-167`

```typescript
export const getStyling = (
  settings: TWorkspaceStateSettings,
  survey: TWorkspaceStateSurvey
): TWorkspaceStyling | TSurveyStyling => {
  if (settings.styling.allowStyleOverwrite) {
    if (!survey.styling?.overwriteThemeStyling) {
      return settings.styling;  // 工作空间允许覆盖，但调查未开启
    }
    return survey.styling;     // 两者都开启，使用调查样式
  }
  return settings.styling;     // 工作空间不允许覆盖
};
```

**真值表**：

| workspace.allowStyleOverwrite | survey.overwriteThemeStyling | 最终使用的样式 |
|------------------------------|-----------------------------|---------------|
| false                        | 任意值                      | workspace 样式 |
| true                         | false / null / undefined    | workspace 样式 |
| true                         | true                        | survey 样式   |

**关键设计意图**：
- `allowStyleOverwrite` 是**工作空间级总开关**，管理员可以完全禁止调查级样式自定义
- `overwriteThemeStyling` 是**调查级开关**，即使工作空间允许，单个调查也可以选择不覆盖

### 3.4 其他配置的继承关系

| 配置项               | 继承来源       | 可覆盖 | 覆盖字段位置          |
|---------------------|---------------|--------|----------------------|
| recontactDays       | workspace     | 是     | survey.recontactDays |
| displayLimit        | workspace     | 是     | survey.displayLimit  |
| displayOption       | workspace     | 是     | survey.displayOption |
| inAppSurveyBranding | workspace     | 否     | 无（全局强制）       |
| recaptchaSiteKey    | workspace     | 否     | 无（全局配置）       |

---

## 四、客户端 SDK 配置管理

### 4.1 Config 单例模式

**文件**：`packages/js-core/src/lib/common/config.ts:6`

```typescript
export class Config {
  private static instance: Config | null = null;
  private config: TConfig | null = null;

  static getInstance(): Config {
    Config.instance ??= new Config();
    return Config.instance;
  }
}
```

### 4.2 配置合并策略

**文件**：`packages/js-core/src/lib/common/config.ts:23-34`

```typescript
public update(newConfig: TConfigUpdateInput): void {
  this.config = {
    ...this.config,
    ...newConfig,
    status: {
      value: newConfig.status?.value ?? "success",
      expiresAt: newConfig.status?.expiresAt ?? null,
    },
  };
  void this.saveToStorage();
}
```

**合并规则**：
1. 浅合并：`{ ...existing, ...new }`
2. 顶层字段：新配置完全覆盖旧配置（如果提供）
3. `status` 对象：特殊合并逻辑
   - `status.value`：新值优先，默认 "success"
   - `status.expiresAt`：新值优先，默认 `null`

### 4.3 本地存储与初始化流程

**文件**：`packages/js-core/src/lib/common/setup.ts:72-329`

**初始化流程**：
```
1. 迁移检查：migrateLocalStorage()
   ├─ 检查旧格式 environmentId → workspaceId
   ├─ 检查旧格式 environment → workspace
   └─ 检查旧格式 project → settings

2. 现有配置检查
   ├─ 配置存在且匹配 → 检查过期时间
   │   ├─ workspace 过期 → 重新 fetchWorkspaceState
   │   └─ user 过期 → 重新 fetch 用户状态
   └─ 配置不存在或不匹配 → 完整初始化
       ├─ fetchWorkspaceState（服务端）
       ├─ 设置默认 userState
       └─ 有 userId 时 fetch 用户状态

3. 过滤调查：filterSurveys(workspace, userState)

4. 更新配置：config.update({ workspace, user, filteredSurveys })
```

### 4.4 过期检查与自动刷新

**文件**：`packages/js-core/src/lib/workspace/state.ts:69-126`

```typescript
export const addWorkspaceStateExpiryCheckListener = (): void => {
  const updateInterval = 1000 * 60;  // 每分钟检查
  if (typeof window !== "undefined" && workspaceSyncIntervalId === null) {
    workspaceSyncIntervalId = window.setInterval(async () => {
      const expiresAt = appConfig.get().workspace.expiresAt;
      if (expiresAt && new Date(expiresAt) >= new Date()) {
        return;  // 未过期，跳过
      }
      // 过期，重新拉取
      const workspace = await fetchWorkspaceState({ ... });
      if (workspace.ok) {
        appConfig.update({
          ...appConfig.get(),
          workspace: workspace.data,
          filteredSurveys: filterSurveys(workspace.data, userState),
        });
      }
    }, updateInterval);
  }
};
```

**过期时间设置**：
- **服务端 Redis 缓存**：1 分钟（`getWorkspaceState`）
- **HTTP 缓存头**：1 分钟（`max-age=60`）
- **SDK 配置过期**：1 小时（`expiresAt`）
- **SDK 检查间隔**：1 分钟

### 4.5 字段重映射（服务端 → 客户端）

**文件**：`packages/js-core/src/lib/workspace/state.ts:43-51`

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

## 五、关键代码位置速查表

| 功能模块               | 文件路径                                                                 | 行号 |
|-----------------------|-------------------------------------------------------------------------|------|
| 服务端数据查询        | `apps/web/app/api/v1/client/[workspaceId]/environment/lib/data.ts`     | 35   |
| 服务端缓存封装        | `apps/web/app/api/v1/client/[workspaceId]/environment/lib/environmentState.ts` | 24 |
| ID 解析兼容           | `apps/web/lib/utils/resolve-client-id.ts`                               | 28   |
| 向后兼容转换          | `apps/web/app/lib/api/api-backwards-compat.ts`                          | 全部 |
| 行为配置优先级        | `packages/js-core/src/lib/survey/widget.ts`                             | 105-108 |
| 样式配置优先级        | `packages/js-core/src/lib/common/utils.ts`                              | 150-167 |
| 客户端配置合并        | `packages/js-core/src/lib/common/config.ts`                             | 23-34 |
| 客户端初始化流程      | `packages/js-core/src/lib/common/setup.ts`                              | 72-329 |
| 配置过期自动刷新      | `packages/js-core/src/lib/workspace/state.ts`                           | 69-126 |
| 字段重映射            | `packages/js-core/src/lib/workspace/state.ts`                           | 43-51 |
| 样式渲染              | `packages/surveys/src/lib/styles.ts`                                    | 58-425 |

---

## 六、优先级总结

### 6.1 总体优先级金字塔

```
┌─────────────────────────────────────┐
│  最高优先级：用户传入参数（runtime） │
│  （如 triggerSurvey 时的 properties）│
├─────────────────────────────────────┤
│  第二级：Survey 级配置               │
│  • workspaceOverwrites（行为）       │
│  • styling（需 allowStyleOverwrite） │
├─────────────────────────────────────┤
│  第三级：Workspace 级配置            │
│  • settings（全局行为）              │
│  • styling（全局样式）               │
├─────────────────────────────────────┤
│  最低优先级：系统默认值              │
│  • 代码硬编码默认值                  │
└─────────────────────────────────────┘
```

### 6.2 样式覆盖决策树

```
是否使用调查级样式？
        │
        ├─ workspace.styling.allowStyleOverwrite = false
        │   → 使用 workspace 样式
        │
        └─ workspace.styling.allowStyleOverwrite = true
            │
            ├─ survey.styling?.overwriteThemeStyling != true
            │   → 使用 workspace 样式
            │
            └─ survey.styling?.overwriteThemeStyling = true
                → 使用 survey 样式
```

### 6.3 缓存层级

```
┌─────────────────────────────────┐
│  最快：SDK 内存（Config 单例）   │  ~0ms
├─────────────────────────────────┤
│  第二：localStorage              │  ~1ms
├─────────────────────────────────┤
│  第三：浏览器 HTTP 缓存          │  ~10ms
├─────────────────────────────────┤
│  第四：CDN 缓存（Cloudflare）    │  ~50ms
├─────────────────────────────────┤
│  第五：Redis 缓存                │  ~1ms + 网络
├─────────────────────────────────┤
│  最慢：数据库查询                │  ~100ms+
└─────────────────────────────────┘
```
