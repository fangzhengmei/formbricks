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

1. **缺少数据校验**：`loadFromLocalStorage()` 直接 `JSON.parse`，没有 schema 校验
2. **错误状态重试机制**：当前 Error 状态只是静默等待过期，没有自动重试
3. **UpdateQueue 失败重试**：网络失败后直接清空队列，应支持重试
4. **存储大小限制**：localStorage 通常 5MB 限制，displays/responses 可能增长过大
5. **缺少加密**：敏感数据（如 contactId）明文存储

### 8.3 边缘情况处理

| 场景 | 处理方式 |
|-----|---------|
| localStorage 被禁用 | `wrapThrows` 捕获异常，SDK 降级为无状态模式 |
| 存储数据格式损坏 | `JSON.parse` 失败视为无配置，重新初始化 |
| 用户清除浏览器缓存 | 视为首次访问，重新拉取环境状态 |
| 隐身模式/隐私浏览 | localStorage 通常可读写，但会话结束后清除 |
| 跨子域访问 | localStorage 按 origin 隔离，需后端配合 |

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
