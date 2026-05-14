# Formbricks JS SDK Session 持久化机制分析

## 1. 概述

Formbricks JS SDK 使用浏览器的 `localStorage` 实现 Session 持久化，存储 SDK 配置、环境状态、用户状态等关键数据。所有数据通过单例模式的 `Config` 类统一管理，确保全局状态一致性。

---

## 2. 存储机制

### 2.1 存储位置与 Key

- **存储介质**：浏览器 `localStorage`
- **存储 Key**：`"formbricks-js"`（定义于 `constants.ts`）
- **相关旧 Key**：
  - `"formbricks-js-website"`（网站 SDK 遗留）
  - `"formbricks-js-app"`（应用 SDK 遗留）

### 2.2 存储数据结构（TConfig）

```typescript
interface TConfig {
  environmentId: string;           // 环境 ID
  appUrl: string;                  // 应用 URL
  environment: TEnvironmentState;  // 环境状态（含问卷列表）
  user: TUserState;                // 用户状态
  filteredSurveys: TEnvironmentStateSurvey[];  // 过滤后的问卷
  status: {                        // SDK 状态
    value: "success" | "error";
    expiresAt: Date | null;
  };
}
```

### 2.3 环境状态（TEnvironmentState）

```typescript
interface TEnvironmentState {
  expiresAt: Date;  // 环境状态过期时间
  data: {
    surveys: TEnvironmentStateSurvey[];      // 问卷列表
    actionClasses: TEnvironmentStateActionClass[];  // 动作类
    project: TEnvironmentStateProject;       // 项目配置
    recaptchaSiteKey?: string;               // reCAPTCHA Key
  };
}
```

### 2.4 用户状态（TUserState）

```typescript
interface TUserState {
  expiresAt: Date | null;  // 用户状态过期时间
  data: {
    userId: string | null;          // 用户 ID
    contactId: string | null;       // 联系人 ID
    segments: string[];             // 用户所属分段
    displays: { surveyId: string; createdAt: Date }[];  // 展示记录
    responses: string[];            // 提交的问卷 ID 列表
    lastDisplayAt: Date | null;     // 最后展示时间
    language?: string;               // 用户语言偏好
  };
}
```

---

## 3. 数据保存流程

### 3.1 Config 单例类（核心）

**文件位置**：`packages/js-core/src/lib/common/config.ts`

```typescript
export class Config {
  private static instance: Config | null = null;
  private config: TConfig | null = null;

  // 构造时自动从 localStorage 加载
  private constructor() {
    const savedConfig = this.loadFromLocalStorage();
    if (savedConfig.ok) {
      this.config = savedConfig.data;
    }
  }

  static getInstance(): Config {
    Config.instance ??= new Config();
    return Config.instance;
  }

  // 更新配置并自动保存
  public update(newConfig: TConfigUpdateInput): void {
    this.config = {
      ...this.config,
      ...newConfig,
      status: {
        value: newConfig.status?.value ?? "success",
        expiresAt: newConfig.status?.expiresAt ?? null,
      },
    };
    void this.saveToStorage();  // ✅ 每次更新自动保存
  }

  // 从 localStorage 加载
  public loadFromLocalStorage(): Result<TConfig> {
    if (typeof window !== "undefined") {
      const savedConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
      if (savedConfig) {
        const parsedConfig = JSON.parse(savedConfig) as TConfig;
        return ok(parsedConfig);
      }
    }
    return err(new Error("No or invalid config in local storage"));
  }

  // 保存到 localStorage
  private saveToStorage(): Result<void> {
    return wrapThrows(() => {
      localStorage.setItem(JS_LOCAL_STORAGE_KEY, JSON.stringify(this.config));
    })();
  }

  // 重置配置（清除 localStorage）
  public resetConfig(): Result<void> {
    this.config = null;
    return wrapThrows(() => {
      localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
    })();
  }
}
```

### 3.2 保存触发时机

1. **SDK 初始化（setup）**：首次加载或配置过期时，拉取最新环境状态和用户状态后保存
2. **用户身份设置（setUserId）**：设置用户 ID 后，通过 UpdateQueue 同步到后端并更新本地状态
3. **属性更新（setAttribute）**：更新用户属性后同步
4. **问卷展示/提交**：更新 displays/responses 后自动保存
5. **语言设置**：更新用户语言偏好后保存

---

## 4. 数据恢复流程

### 4.1 初始化恢复（setup 函数）

**文件位置**：`packages/js-core/src/lib/common/setup.ts`

```typescript
export const setup = async (configInput: TConfigInput): Promise<Result<void>> => {
  let config = Config.getInstance();  // ✅ 构造时自动从 localStorage 恢复

  // 1. 数据迁移（处理旧版本格式）
  const { changed, newState } = migrateLocalStorage();
  if (changed && newState) {
    config.resetConfig();
    config = Config.getInstance();
    if (!newState.user?.data?.userId) {
      config.update(newState);  // 应用迁移后的配置
    }
  }

  // 2. 检查现有配置有效性
  let existingConfig: TConfig | undefined;
  try {
    existingConfig = config.get();
    logger.debug("Found existing configuration.");
  } catch {
    logger.debug("No existing configuration found.");
  }

  // 3. 检查 Error 状态是否过期
  if (existingConfig?.status.value === "error") {
    const expiresAt = existingConfig.status.expiresAt;
    if (expiresAt && !isNowExpired(new Date(expiresAt))) {
      console.error("🧱 Formbricks - Error state is not expired, skipping initialization");
      return okVoid();
    }
  }

  // 4. 检查环境/用户状态是否过期
  if (existingConfig?.environment &&
      existingConfig.environmentId === configInput.environmentId &&
      existingConfig.appUrl === configInput.appUrl) {
    
    const environmentStateExpiresAt = new Date(existingConfig.environment.expiresAt);
    if (isNowExpired(environmentStateExpiresAt)) {
      // 环境状态过期，重新从后端拉取
      const environmentStateResponse = await fetchEnvironmentState(...);
      // ... 更新后保存
    }

    if (existingConfig.user.expiresAt && isNowExpired(new Date(existingConfig.user.expiresAt))) {
      // 用户状态过期，重新同步
      if (userState.data.userId) {
        const updatesResponse = await sendUpdatesToBackend(...);
        // ... 更新后保存
      }
    }
  }
};
```

### 4.2 本地存储数据迁移

```typescript
const migrateLocalStorage = (): { changed: boolean; newState?: TConfig } => {
  const existingConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
  if (existingConfig) {
    const parsedConfig = JSON.parse(existingConfig) as TLegacyConfig;
    
    // 检测旧格式（有 environmentState 字段）
    if (parsedConfig.environmentState) {
      const { apiHost, environmentState, personState, attributes, ...rest } = parsedConfig;
      
      // 转换为新格式
      const newLocalStorageConfig: TConfig = {
        ...rest,
        ...(apiHost && { appUrl: apiHost }),
        environment: environmentState,
        ...(personState && {
          user: {
            ...personState,
            data: {
              ...personState.data,
              ...(attributes?.language && { language: attributes.language as string }),
            },
          },
        }),
      };
      return { changed: true, newState: newLocalStorageConfig };
    }
  }
  return { changed: false };
};
```

---

## 5. 失效机制

### 5.1 过期检查函数

**文件位置**：`packages/js-core/src/lib/common/utils.ts`

```typescript
export const isNowExpired = (expirationDate: Date): boolean => 
  new Date() >= expirationDate;
```

### 5.2 各类数据的过期策略

| 数据类型 | 过期时间 | 过期后行为 |
|---------|---------|-----------|
| **环境状态** | 由后端返回（通常几小时） | 重新从后端拉取完整环境状态 |
| **已登录用户状态** | 30 分钟 | 调用 `sendUpdatesToBackend` 同步最新用户状态 |
| **未登录用户状态** | `null`（永不过期） | 使用默认空状态 |
| **Error 状态** | 10 分钟 | 过期后允许重新初始化 |

### 5.3 用户状态自动续期

**文件位置**：`packages/js-core/src/lib/user/state.ts`

```typescript
let userStateSyncIntervalId: number | null = null;

export const addUserStateExpiryCheckListener = (): void => {
  const config = Config.getInstance();
  const updateInterval = 1000 * 60;  // 每 60 秒检查一次

  if (typeof window !== "undefined" && userStateSyncIntervalId === null) {
    const intervalHandler = (): void => {
      const userId = config.get().user.data.userId;
      if (!userId) return;

      // ✅ 延长用户状态有效期 30 分钟
      config.update({
        ...config.get(),
        user: {
          ...config.get().user,
          expiresAt: new Date(new Date().getTime() + 1000 * 60 * 30),
        },
      });
    };

    userStateSyncIntervalId = setInterval(intervalHandler, updateInterval) as unknown as number;
  }
};
```

**工作原理**：
- 每 60 秒检查一次是否有已登录用户
- 如果有用户，自动将 `user.expiresAt` 延长 30 分钟
- 确保用户在活跃期间不会因状态过期而需要频繁重新同步

### 5.4 主动失效（登出）

**文件位置**：`packages/js-core/src/lib/user/user.ts`

```typescript
export const logout = (): Result<void> => {
  try {
    const logger = Logger.getInstance();
    logger.debug("Logging out and cleaning user state");
    tearDown();  // ✅ 调用清理函数
    return okVoid();
  } catch {
    return { ok: false, error: new Error("Failed to logout") };
  }
};

// setup.ts 中的 tearDown
export const tearDown = (): void => {
  const logger = Logger.getInstance();
  const appConfig = Config.getInstance();
  const { environment } = appConfig.get();
  
  // 使用未登录用户的默认状态过滤问卷
  const filteredSurveys = filterSurveys(environment, DEFAULT_USER_STATE_NO_USER_ID);
  
  logger.debug("Setting user state to default");
  
  // ✅ 重置为默认用户状态
  appConfig.update({
    ...appConfig.get(),
    user: DEFAULT_USER_STATE_NO_USER_ID,
    filteredSurveys,
  });
  
  closeSurvey();  // 关闭当前展示的问卷
};
```

### 5.5 错误状态处理

```typescript
export const putFormbricksInErrorState = (formbricksConfig: Config): void => {
  const logger = Logger.getInstance();
  if (getIsDebug()) {
    logger.debug("Not putting formbricks in error state because debug mode is active");
    return;
  }

  logger.debug("Putting formbricks in error state");
  formbricksConfig.update({
    ...formbricksConfig.get(),
    status: {
      value: "error",
      expiresAt: new Date(new Date().getTime() + 10 * 60000),  // ✅ 10 分钟后过期
    },
  });
  tearDown();
};
```

---

## 6. UpdateQueue 异步更新队列

### 6.1 队列机制

**文件位置**：`packages/js-core/src/lib/user/update-queue.ts`

```typescript
export class UpdateQueue {
  private static instance: UpdateQueue | null = null;
  private updates: TUpdates | null = null;
  private debounceTimeout: NodeJS.Timeout | null = null;
  private pendingFlush: Promise<void> | null = null;
  private readonly DEBOUNCE_DELAY = 500;       // 500ms 防抖
  private readonly PENDING_WORK_TIMEOUT = 5000; // 5s 超时

  public updateUserId(userId: string): void { /* ... */ }
  public updateAttributes(attributes: TAttributes): void { /* ... */ }

  public async processUpdates(): Promise<void> {
    // 防抖处理：500ms 内多次调用合并为一次
    // 异步发送到后端
    // 更新本地用户状态
    // 保存到 localStorage
  }
}
```

### 6.2 关键特性

1. **防抖合并**：500ms 内的多次 `setUserId`/`setAttribute` 合并为一次请求
2. **异步处理**：不阻塞主线程，后台静默同步
3. **超时保护**：5 秒超时，防止网络问题卡住
4. **本地优先**：无 userId 时，language 属性直接保存到本地
5. **错误处理**：网络失败时保留在队列中，后续重试（当前实现为清空队列）

---

## 7. 完整生命周期流程图

```
浏览器刷新/页面加载
        ↓
Config.getInstance() 构造
        ↓
┌─────────────────────────┐
│ loadFromLocalStorage()  │  从 localStorage 恢复
└─────────────────────────┘
        ↓
┌─────────────────────────┐
│ migrateLocalStorage()   │  数据格式迁移（如需要）
└─────────────────────────┘
        ↓
setup() 函数执行
        ↓
┌─────────────────────────┐
│ 检查 status 是否为 error │
│ 检查环境状态是否过期     │
│ 检查用户状态是否过期     │
└─────────────────────────┘
        ├─ 过期 → 从后端拉取最新数据 → update() 保存
        └─ 有效 → 直接使用
        ↓
┌─────────────────────────┐
│ addUserStateExpiryCheck │  启动自动续期定时器
└─────────────────────────┘
        ↓
正常运行（展示问卷、收集反馈）
        ↓
用户操作触发状态变更（setUserId/setAttribute）
        ↓
┌─────────────────────────┐
│ UpdateQueue 入队        │
│ 500ms 防抖合并          │
│ 发送到后端              │
│ 更新本地 user state     │
│ 保存到 localStorage     │
└─────────────────────────┘
        ↓
用户登出 → tearDown() → 重置为默认用户状态
```

---

## 8. 关键设计要点

### 8.1 优点

1. **单例模式**：确保全局配置一致，避免多实例状态不同步
2. **自动保存**：`Config.update()` 自动持久化，无需手动调用
3. **防抖优化**：UpdateQueue 合并频繁更新，减少网络请求
4. **渐进式失效**：环境状态和用户状态独立过期，粒度精细
5. **向后兼容**：内置数据迁移逻辑，平滑升级 SDK 版本
6. **本地优先**：无网络时也能基于缓存工作

### 8.2 潜在改进点

1. **🔥 严重 Bug：两处 `JSON.parse` 均无异常捕获**：`config.ts` 和 `setup.ts` 各有一处，数据损坏会导致 SDK 完全无法初始化（详见 8.4 节完整分析）
2. **缺少数据 schema 校验**：即使 JSON 格式正确，也可能缺少必要字段，应增加基本结构校验
3. **setup 缺少外层兜底异常捕获**：任何初始化阶段的异常都会导致 SDK 彻底中断
4. **错误状态重试机制**：当前 Error 状态只是静默等待过期，没有自动重试
5. **UpdateQueue 失败重试**：网络失败后直接清空队列，应支持重试
6. **存储大小限制**：localStorage 通常 5MB 限制，displays/responses 可能增长过大
7. **缺少加密**：敏感数据（如 contactId）明文存储

### 8.3 边缘情况处理

| 场景 | 处理方式 |
|-----|---------|
| localStorage 被禁用 | `saveToStorage()`/`resetConfig()` 使用 `wrapThrows` 捕获异常，但 `loadFromLocalStorage()` 没有，详见下文 |
| 存储数据格式损坏 | `JSON.parse` 失败**直接抛出未捕获异常**，导致 SDK 初始化中断（详见下文分析） |
| 用户清除浏览器缓存 | 视为首次访问，重新拉取环境状态 |
| 隐身模式/隐私浏览 | localStorage 通常可读写，但会话结束后清除 |
| 跨子域访问 | localStorage 按 origin 隔离，需后端配合 |

---

### 8.4 问题诊断：事实与推测分区

---

#### 📌 分区说明
| 标识 | 含义 | 是否作为修复依据 |
|-----|------|----------------|
| ✅ 【事实】 | 代码层面可确认的风险，有明确的代码证据 | **是**，必须修复 |
| ⚠️ 【推测】 | 理论上可能发生，但无明确代码证据 | 否，仅作了解 |

---

#### ✅ 【事实风险 1】`config.ts` 的 `loadFromLocalStorage()` 无异常捕获

**代码证据**（`packages/js-core/src/lib/common/config.ts` 第 43-56 行）：

```typescript
public loadFromLocalStorage(): Result<TConfig> {
  if (typeof window !== "undefined") {
    const savedConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
    if (savedConfig) {
      const parsedConfig = JSON.parse(savedConfig) as TConfig;  // ❌ 无 try-catch！【事实】
      return ok(parsedConfig);
    }
  }
  return err(new Error("No or invalid config in local storage"));
}
```

**触发条件【事实】**：任何情况下调用 `Config.getInstance()` 时，只要 localStorage 有数据且不是合法 JSON

**异常传播路径【事实】**：

```
localStorage.getItem() 返回非空字符串【事实】
        ↓
JSON.parse() 抛出 SyntaxError【事实：无效 JSON 必然抛出】
        ↓
loadFromLocalStorage() 无 try-catch【事实】
        ↓
Config 构造函数无 try-catch【事实】
        ↓
Config.getInstance() 抛出异常【事实】
        ↓
setup() 第 75 行调用 Config.getInstance() 时无 try-catch【事实】
        ↓
整个 SDK 初始化中断【必然结果】
```

**风险等级**：🔴 阻塞级 - 100% 触发，完全无法初始化

---

#### ✅ 【事实风险 2】`setup.ts` 的 `migrateLocalStorage()` 无异常捕获

**代码证据**（`packages/js-core/src/lib/common/setup.ts` 第 28-63 行）：

```typescript
const migrateLocalStorage = (): { changed: boolean; newState?: TConfig } => {
  const existingConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);

  if (existingConfig) {
    const parsedConfig = JSON.parse(existingConfig) as TLegacyConfig;  // ❌ 无 try-catch【事实】

    if (parsedConfig.environmentState) {
      const { apiHost, environmentState, personState, attributes, ...rest } = parsedConfig;
      // ❌ 解构属性也可能抛 TypeError【事实：属性不存在或类型不对时】
    }
  }
  return { changed: false };
};
```

**触发条件【事实】**：localStorage 有数据但不是合法 JSON，或 JSON 合法但结构异常导致解构失败

**异常传播路径【事实】**：

```
localStorage 有数据但格式异常
        ↓
风险点 1 可能侥幸通过（如刚好是合法但异常的旧格式）
        ↓
setup() 第 77 行调用 migrateLocalStorage()
        ↓
JSON.parse() 或属性解构时抛出异常【事实】
        ↓
migrateLocalStorage() 无 try-catch【事实】
        ↓
setup() 函数执行中断【必然结果】
```

**风险等级**：🔴 高 - 触发概率较低，但一旦发生同样完全中断

---

#### ⚠️ 【推测区】理论上可能的边缘场景（不作为修复依据）

**推测场景 1：两次 getItem 返回不同内容**
- 推测内容：`Config.getInstance()` 和 `migrateLocalStorage()` 两次读取 localStorage 可能拿到不同值
- 推测依据：理论上可能有浏览器扩展或异常情况干预
- **但标准浏览器环境下 localStorage 是同步且原子的，此场景极难复现**
- **不作为修复依据，仅作了解**

**推测场景 2：其他未发现的初始化异常点**
- 推测内容：除了两处 JSON.parse，初始化流程中可能还有其他未捕获的异常
- 建议：通过第 3 步的兜底 try-catch 统一防护

---

#### 两个事实风险点对比

| 对比项 | 事实风险 1 (config.ts) | 事实风险 2 (setup.ts) |
|-------|---------------------|---------------------|
| **执行时机** | 更早（setup 第 75 行之前） | 稍晚（setup 第 77 行） |
| **触发概率** | 高（任何数据损坏都会触发） | 低（仅影响升级用户 + 结构异常） |
| **严重性** | 🔴 阻塞级 - 100% 无法初始化 | 🔴 高 - 同样初始化中断 |
| **返回类型设计** | 声明返回 `Result<T>` 但实际抛异常（违背设计约定） | 直接返回普通对象，无错误处理约定 |
| **异常类型** | `SyntaxError` (JSON.parse) | `SyntaxError` + 可能的 `TypeError` |
| **代码证据** | ✅ 完全确认 | ✅ 完全确认 |

---

#### 对比：其他方法的异常处理【事实】

`saveToStorage()` 和 `resetConfig()` 使用了 `wrapThrows` 正确捕获异常：

```typescript
private saveToStorage(): Result<void> {
  return wrapThrows(() => {
    localStorage.setItem(JS_LOCAL_STORAGE_KEY, JSON.stringify(this.config));
  })();  // ✅ 正确使用 wrapThrows【事实】
}
```

但两处 `loadFromLocalStorage` / `migrateLocalStorage` 都没有遵循相同的错误处理模式【事实】。

---

### 8.5 可直接执行的整改方案

#### 🔧 修复顺序与依赖关系（严格按此顺序执行）

| 序号 | 修复内容 | 依赖关系 | 跳过的风险 | 优先级 |
|-----|---------|---------|-----------|--------|
| **第 1 步** | 修复 `config.ts` 的 `loadFromLocalStorage()` | 无前置依赖，可独立上线 | ❌ 99% 的数据损坏场景会导致 SDK 100% 崩溃 | 🔴 阻塞级 - 必须立即修 |
| **第 2 步** | 修复 `setup.ts` 的 `migrateLocalStorage()` | 不依赖第 1 步，可并行 | ❌ 旧版本升级时可能出现迁移崩溃，影响存量用户 | 🟠 高 - 建议 24h 内修 |
| **第 3 步** | 在 `setup()` 最外层增加兜底 try-catch | 建议在 1+2 之后做 | ❌ 未来新增代码引入异常时 SDK 会彻底崩溃，无降级路径 | 🟡 中 - 建议 72h 内修 |

---

#### ✅ 第 1 步：修复 `config.ts` 的 `loadFromLocalStorage()`

**修改文件**：`packages/js-core/src/lib/common/config.ts`

**代码修改**：

```typescript
public loadFromLocalStorage(): Result<TConfig> {
  if (typeof window !== "undefined") {
    const savedConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
    if (savedConfig) {
      try {
        const parsedConfig = JSON.parse(savedConfig) as TConfig;
        // TODO: validate config
        return ok(parsedConfig);
      } catch (error) {
        // 数据损坏，主动清除，让后续流程重新初始化
        try {
          localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
        } catch {}
        return err(new Error("Corrupted config in local storage, cleared"));
      }
    }
  }
  return err(new Error("No or invalid config in local storage"));
}
```

**验证要点**：
1. 手动在 localStorage 中写入非法 JSON：`localStorage.setItem('formbricks-js', '{invalid')`
2. 刷新页面，确认 SDK 能正常初始化（不崩溃）
3. 确认损坏数据已被自动清除

---

#### ✅ 第 2 步：修复 `setup.ts` 的 `migrateLocalStorage()`

**修改文件**：`packages/js-core/src/lib/common/setup.ts`

**代码修改**：

```typescript
const migrateLocalStorage = (): { changed: boolean; newState?: TConfig } => {
  const existingConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);

  if (existingConfig) {
    try {
      const parsedConfig = JSON.parse(existingConfig) as TLegacyConfig;

      // Check if we need to migrate (if it has environmentState, it's old format)
      if (parsedConfig.environmentState) {
        const { apiHost, environmentState, personState, attributes, ...rest } = parsedConfig;

        const newLocalStorageConfig: TConfig = {
          ...rest,
          ...(apiHost && { appUrl: apiHost }),
          environment: environmentState,
          ...(personState && {
            user: {
              ...personState,
              data: {
                ...personState.data,
                ...(attributes?.language && { language: attributes.language as string }),
              },
            },
          }),
        };

        return {
          changed: true,
          newState: newLocalStorageConfig,
        };
      }
    } catch (error) {
      // 迁移时出错，静默返回未变更，让后续流程自然处理
      console.warn("🧱 Formbricks - Failed to migrate config, will reset to fresh state");
      try {
        localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
      } catch {}
    }
  }

  return { changed: false };
};
```

**验证要点**：
1. 模拟旧版本数据：写入含 `environmentState` 但结构异常的 JSON
2. 刷新页面，确认迁移失败后 SDK 能降级初始化（不崩溃）
3. 确认损坏数据已被自动清除

---

#### ✅ 第 3 步：在 `setup()` 最外层增加兜底 try-catch

**修改文件**：`packages/js-core/src/lib/common/setup.ts`

**代码修改**：

```typescript
export const setup = async (
  configInput: TConfigInput | (TConfigInput & { userId: string; attributes: Record<string, string> })
): Promise<Result<void, MissingFieldError | NetworkError | MissingPersonError>> => {
  try {  // 最外层兜底
    const isDebug = getIsDebug();
    const logger = Logger.getInstance();

    if (isDebug) {
      logger.configure({ logLevel: "debug" });
    }

    let config = Config.getInstance();

    // ... 原有的全部 setup 代码 保持不变 ...

    setIsSetup(true);
    logger.debug("Set up complete");

    return okVoid();
  } catch (error) {
    console.error("🧱 Formbricks - Fatal error during SDK initialization:", error);

    // 尝试彻底重置为干净状态
    try {
      localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
    } catch {}

    // 返回一个通用错误，但至少不会让 JS 线程崩溃
    return err({
      code: "initialization_error",
      message: "SDK initialization failed",
      status: 500,
      url: new URL(window.location.href),
      responseMessage: error instanceof Error ? error.message : "Unknown error",
    });
  }
};
```

**验证要点**：
1. 在 `Config.getInstance()` 中故意抛出异常
2. 确认 SDK 不会崩溃，而是返回 `initialization_error`
3. 确认 localStorage 数据已被清除
4. 确认业务方可以根据错误码做降级处理

---

#### （可选增强）第 4 步：增加数据完整性校验

**适用场景**：生产环境需要极高健壮性保证，或已观察到结构异常导致的运行时错误

**依赖关系**：必须在第 1 步完成后才能做

**代码修改参考**：见 8.5 节原文

---

#### 📋 验收标准

| 序号 | 验收项 | 通过标准 |
|-----|--------|---------|
| 1 | localStorage 数据完全损坏 | SDK 正常初始化，数据被自动清除 |
| 2 | localStorage 数据是旧格式但合法 | 正常迁移或降级初始化，不崩溃 |
| 3 | localStorage 数据是旧格式但结构异常 | 迁移失败后降级初始化，不崩溃 |
| 4 | 初始化流程任意位置抛出异常 | 不崩溃，返回 `initialization_error`，业务方可降级 |
| 5 | 正常场景（数据完好） | 功能不受影响，性能无明显下降 |

---

## 9. 相关文件索引

| 文件 | 功能 |
|-----|------|
| `packages/js-core/src/lib/common/config.ts` | Config 单例类，核心存储管理 |
| `packages/js-core/src/lib/common/setup.ts` | SDK 初始化，状态恢复与过期检查 |
| `packages/js-core/src/lib/common/constants.ts` | 存储 Key 定义 |
| `packages/js-core/src/lib/user/state.ts` | 用户状态管理，自动续期 |
| `packages/js-core/src/lib/user/update-queue.ts` | 异步更新队列 |
| `packages/js-core/src/lib/user/update.ts` | 后端同步逻辑 |
| `packages/js-core/src/lib/user/user.ts` | setUserId/logout API |
| `packages/js-core/src/types/config.ts` | 所有类型定义 |
