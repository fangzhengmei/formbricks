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

### 8.4 本地存储数据损坏的 Bug 分析

当前代码存在 **两处独立的 `JSON.parse` 未捕获异常风险**，分别在不同执行路径上：

---

#### 风险点 1：`config.ts` 中的 `loadFromLocalStorage()`

**触发条件**：任何情况下调用 `Config.getInstance()` 时，只要 localStorage 有数据且格式损坏

**问题代码证据**（`packages/js-core/src/lib/common/config.ts` 第 43-56 行）：

```typescript
public loadFromLocalStorage(): Result<TConfig> {
  if (typeof window !== "undefined") {
    const savedConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
    if (savedConfig) {
      // TODO: validate config
      // This is a hack to get around the fact that we don't have a proper
      // way to validate the config yet.
      const parsedConfig = JSON.parse(savedConfig) as TConfig;  // ❌ 无 try-catch！
      return ok(parsedConfig);
    }
  }

  return err(new Error("No or invalid config in local storage"));
}
```

**异常传播路径**：

```
localStorage 数据损坏（任何格式错误）
        ↓
JSON.parse() 抛出 SyntaxError
        ↓
loadFromLocalStorage() 未捕获，异常向上抛出
        ↓
Config 构造函数未捕获，异常继续向上
        ↓
Config.getInstance() 抛出异常
        ↓
setup() 函数第 75 行调用 Config.getInstance() 时 ❌ 无 try-catch 包裹
        ↓
整个 SDK 初始化中断，后续代码无法执行
```

**实际影响**：
- 只要 localStorage 存在损坏数据，SDK **100% 无法初始化**
- 即使业务方想降级使用（如不加载历史配置直接重新初始化）也不可能
- 这是**更早执行、更致命**的风险点

---

#### 风险点 2：`setup.ts` 中的 `migrateLocalStorage()`

**触发条件**：localStorage 有数据、且数据刚好是**合法的旧格式 JSON**（含 `environmentState` 字段需要迁移），但迁移过程中访问的字段存在异常

**问题代码证据**（`packages/js-core/src/lib/common/setup.ts` 第 28-63 行）：

```typescript
const migrateLocalStorage = (): { changed: boolean; newState?: TConfig } => {
  const existingConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);

  if (existingConfig) {
    const parsedConfig = JSON.parse(existingConfig) as TLegacyConfig;  // ❌ 同样无 try-catch！

    // Check if we need to migrate (if it has environmentState, it's old format)
    if (parsedConfig.environmentState) {
      // ... 迁移逻辑：解构 apiHost, environmentState, personState, attributes
      // ❌ 如果这些字段类型异常（如 personState 不是 object），解构时也可能抛异常
    }
  }

  return { changed: false };
};
```

**异常传播路径**：

```
localStorage 数据是合法 JSON，但结构异常（需迁移的旧格式）
        ↓
风险点 1 成功通过：JSON 格式合法，Config.getInstance() 成功返回
        ↓
setup() 第 77 行调用 migrateLocalStorage()
        ↓
migrateLocalStorage() 再次读取 localStorage + JSON.parse
        ↓
JSON.parse() 成功，但后续访问属性时可能因结构异常抛出 TypeError
        ↓
（⚠️ 补充说明：关于「两次 getItem 返回不同内容」的竞态场景，目前仅为理论推测，
  标准浏览器环境下 localStorage 是同步且原子的，除非有浏览器 Bug 或扩展干预）
        ↓
异常未被捕获 → setup() 函数执行中断 ❌
```

**实际影响**：
- 触发场景：旧版本 SDK 升级到新版本时，本地存储的旧格式数据结构异常
- 触发概率：低（仅影响升级用户，且要求 JSON 合法但结构异常）
- 但一旦发生，同样导致 SDK 初始化中断
- **代码层面的事实风险**：`JSON.parse` 确实没有 try-catch，这是确定的 Bug

---

#### 两个风险点的对比

| 对比项 | 风险点 1 (config.ts) | 风险点 2 (setup.ts) |
|-------|---------------------|---------------------|
| **执行时机** | 更早（setup 第 75 行之前） | 稍晚（setup 第 77 行） |
| **触发概率** | 高（任何数据损坏都会触发） | 低（需要特定竞态条件） |
| **严重性** | 🔴 最高 - 完全无法初始化 | 🔴 高 - 同样初始化中断 |
| **返回类型设计** | 声明返回 `Result<T>` 但实际会抛异常 | 直接返回普通对象，无任何错误处理约定 |
| **重复读取** | 读一次 localStorage | 再次读 localStorage（可能不一致） |

#### 对比：其他方法的异常处理

`saveToStorage()` 和 `resetConfig()` 使用了 `wrapThrows` 正确捕获异常：

```typescript
private saveToStorage(): Result<void> {
  return wrapThrows(() => {
    localStorage.setItem(JS_LOCAL_STORAGE_KEY, JSON.stringify(this.config));
  })();  // ✅ 正确使用 wrapThrows
}
```

但两处 `loadFromLocalStorage` / `migrateLocalStorage` 都没有遵循相同的错误处理模式。

---

### 8.5 可执行的改进建议

#### 建议 1：修复 JSON.parse 异常捕获（最高优先级）

```typescript
// packages/js-core/src/lib/common/config.ts
public loadFromLocalStorage(): Result<TConfig> {
  if (typeof window !== "undefined") {
    const savedConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
    if (savedConfig) {
      try {
        const parsedConfig = JSON.parse(savedConfig) as TConfig;
        // TODO: validate config
        return ok(parsedConfig);
      } catch (error) {
        // 数据损坏，清除损坏的存储项，返回错误
        localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
        return err(new Error("Corrupted config in local storage, cleared"));
      }
    }
  }
  return err(new Error("No or invalid config in local storage"));
}
```

#### 建议 2：在 setup 入口增加兜底 try-catch

```typescript
// packages/js-core/src/lib/common/setup.ts
export const setup = async (...): Promise<Result<...>> => {
  try {  // ✅ 增加外层兜底
    const isDebug = getIsDebug();
    const logger = Logger.getInstance();
    // ... 现有代码
  } catch (error) {
    console.error("🧱 Formbricks - Fatal error during setup:", error);
    // 尝试重置为干净状态
    try {
      localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
    } catch {}
    return err({
      code: "initialization_error",
      message: "SDK initialization failed due to corrupted state",
      status: 500,
      url: new URL(window.location.href),
      responseMessage: error instanceof Error ? error.message : "Unknown error",
    });
  }
};
```

#### 建议 3：增加数据完整性校验

```typescript
// 简单的 schema 校验（可逐步完善）
const isValidConfig = (obj: unknown): obj is TConfig => {
  if (!obj || typeof obj !== "object") return false;
  const config = obj as Record<string, unknown>;
  return (
    typeof config.environmentId === "string" &&
    typeof config.appUrl === "string" &&
    typeof config.environment === "object" &&
    typeof config.user === "object"
  );
};

// 在 loadFromLocalStorage 中使用：
const parsedConfig = JSON.parse(savedConfig);
if (!isValidConfig(parsedConfig)) {
  localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
  return err(new Error("Invalid config schema"));
}
return ok(parsedConfig as TConfig);
```

#### 建议 4：损坏数据时自动降级并上报

```typescript
// 在 catch 块中
catch (error) {
  logger.warn("Corrupted config detected, resetting to fresh state");
  localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
  // 可选：上报错误到监控系统
  if (typeof window !== "undefined" && (window as any).formbricksOnError) {
    (window as any).formbricksOnError(error, "config_corrupted");
  }
  return err(new Error("Config corrupted, reset"));
}
```

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
