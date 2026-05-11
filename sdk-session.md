# Formbricks SDK 会话状态保留机制详解

## 概述

Formbricks SDK 提供了完善的会话状态保留机制，确保访客刷新页面后能够继续上次未填完的问卷。本文档将从 **SDK 初始化**、**状态本地存储**、**断点续填** 三个核心环节详细解析其实现原理。

---

## 一、SDK 初始化

### 1.1 初始化入口

SDK 初始化通过 `formbricks.setup()` 方法触发，入口位于 `packages/js-core/src/index.ts`。初始化流程会先检查是否已有有效的缓存配置，避免重复初始化。

```typescript
// packages/js-core/src/index.ts:14-48
const setup = async (setupConfig: TConfigInput): Promise<void> => {
  // 命令队列确保初始化顺序
  await queue.add(Setup.setup, CommandType.Setup, false, setupConfig);
  await queue.wait();
  
  // 延迟检查页面 URL，允许用户在 setup() 后同步调用 setUserId 等方法
  setTimeout(() => {
    void checkPageUrl();
  }, 0);
};
```

### 1.2 核心初始化流程

初始化的核心逻辑在 `packages/js-core/src/lib/common/setup.ts` 中实现，包含以下关键步骤：

#### 步骤 1：LocalStorage 配置迁移

SDK 会首先检查 localStorage 中是否存在旧版本配置，并自动迁移到新格式：

```typescript
// packages/js-core/src/lib/common/setup.ts:28-63
const migrateLocalStorage = (): { changed: boolean; newState?: TConfig } => {
  const existingConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
  
  if (existingConfig) {
    const parsedConfig = JSON.parse(existingConfig) as TLegacyConfig;
    
    // 检查是否需要迁移（旧格式包含 environmentState）
    if (parsedConfig.environmentState) {
      // 迁移到新结构：apiHost -> appUrl, environmentState -> environment, personState -> user
      const newLocalStorageConfig: TConfig = {
        ...rest,
        ...(apiHost && { appUrl: apiHost }),
        environment: environmentState,
        ...(personState && { user: { ...personState, data: { ... } } }),
      };
      return { changed: true, newState: newLocalStorageConfig };
    }
  }
  return { changed: false };
};
```

#### 步骤 2：检查已初始化状态

```typescript
// packages/js-core/src/lib/common/setup.ts:91-94
if (getIsSetup()) {
  logger.debug("Already set up, skipping setup.");
  return okVoid();
}
```

#### 步骤 3：加载已有配置

```typescript
// packages/js-core/src/lib/common/setup.ts:96-102
let existingConfig: TConfig | undefined;
try {
  existingConfig = config.get(); // 从 Config 单例获取，Config 构造时自动从 localStorage 加载
  logger.debug("Found existing configuration.");
} catch {
  logger.debug("No existing configuration found.");
}
```

#### 步骤 4：错误状态检查

```typescript
// packages/js-core/src/lib/common/setup.ts:104-123
if (existingConfig?.status.value === "error") {
  const expiresAt = existingConfig.status.expiresAt;
  if (expiresAt && !isNowExpired(new Date(expiresAt))) {
    // 错误状态未过期，跳过初始化（防止无限重试）
    console.error("🧱 Formbricks - Error state is not expired, skipping initialization");
    return okVoid();
  }
  // 错误状态已过期，继续初始化
}
```

#### 步骤 5：配置一致性检查

```typescript
// packages/js-core/src/lib/common/setup.ts:144-148
if (
  existingConfig?.environment &&
  existingConfig.environmentId === configInput.environmentId &&
  existingConfig.appUrl === configInput.appUrl
) {
  // 配置匹配，检查环境状态和用户状态是否过期
  // 过期则重新同步，未过期则直接使用缓存
}
```

#### 步骤 6：获取环境状态与用户状态

- **环境状态**：从 `fetchEnvironmentState()` 获取，包含可用问卷列表、样式配置等
- **用户状态**：包含用户 ID、展示历史、响应历史等

```typescript
// packages/js-core/src/lib/common/setup.ts:263-304
// 首次初始化时
const environmentStateResponse = await fetchEnvironmentState({
  appUrl: configInput.appUrl,
  environmentId: configInput.environmentId,
});

let userState: TUserState = DEFAULT_USER_STATE_NO_USER_ID;

// 如果提供了 userId，则获取用户状态
if ("userId" in configInput && configInput.userId) {
  const updatesResponse = await sendUpdatesToBackend({ ... });
  if (updatesResponse.ok) {
    userState = updatesResponse.data.state;
  }
}

// 过滤问卷（基于展示选项、重联系天数、用户分段）
const filteredSurveys = filterSurveys(environmentState, userState);

// 更新配置并保存到 localStorage
config.update({
  appUrl: configInput.appUrl,
  environmentId: configInput.environmentId,
  user: userState,
  environment: environmentState,
  filteredSurveys,
});
```

### 1.3 Config 单例与 localStorage 持久化

`Config` 类负责管理 SDK 配置，采用单例模式，构造时自动从 localStorage 加载：

```typescript
// packages/js-core/src/lib/common/config.ts:6-72
export class Config {
  private static instance: Config | null = null;
  private config: TConfig | null = null;

  private constructor() {
    // 构造时自动从 localStorage 加载
    const savedConfig = this.loadFromLocalStorage();
    if (savedConfig.ok) {
      this.config = savedConfig.data;
    }
  }

  public update(newConfig: TConfigUpdateInput): void {
    this.config = { ...this.config, ...newConfig, status: { ... } };
    void this.saveToStorage(); // 自动保存到 localStorage
  }

  public loadFromLocalStorage(): Result<TConfig> {
    if (typeof window !== "undefined") {
      const savedConfig = localStorage.getItem(JS_LOCAL_STORAGE_KEY);
      if (savedConfig) {
        return ok(JSON.parse(savedConfig) as TConfig);
      }
    }
    return err(new Error("No or invalid config in local storage"));
  }

  private saveToStorage(): Result<void> {
    return wrapThrows(() => {
      localStorage.setItem(JS_LOCAL_STORAGE_KEY, JSON.stringify(this.config));
    })();
  }

  public resetConfig(): Result<void> {
    this.config = null;
    return wrapThrows(() => {
      localStorage.removeItem(JS_LOCAL_STORAGE_KEY);
    })();
  }
}
```

**localStorage 存储键名**：`"formbricks-js"`（定义于 `packages/js-core/src/lib/common/constants.ts:1`）

---

## 二、状态本地存储

Formbricks SDK 使用 **两层存储架构** 来管理不同类型的状态：

| 存储层 | 用途 | 存储介质 | 数据类型 |
|--------|------|----------|----------|
| Config 层 | SDK 配置、环境状态、用户状态 | localStorage | 全局配置 |
| 问卷进度层 | 问卷填写进度、响应数据、离线响应 | IndexedDB | 问卷级数据 |

### 2.1 Config 层存储（localStorage）

**存储位置**：`packages/js-core/src/lib/common/config.ts`

**存储内容结构**：

```typescript
// localStorage key: "formbricks-js"
interface TConfig {
  appUrl: string;              // Formbricks 服务地址
  environmentId: string;       // 环境 ID
  environment: TEnvironmentState;  // 环境状态（问卷列表、样式等）
  user: TUserState;            // 用户状态
  filteredSurveys: TEnvironmentStateSurvey[];  // 过滤后的可展示问卷
  status: {
    value: "success" | "error";
    expiresAt: Date | null;
  };
}

interface TEnvironmentState {
  data: {
    surveys: TEnvironmentStateSurvey[];  // 可用问卷列表
    project: TEnvironmentStateProject;   // 项目配置
    actionClasses: TEnvironmentStateActionClass[];  // 触发器配置
  };
  expiresAt: string;  // 环境状态过期时间
}

interface TUserState {
  data: {
    userId: string | null;
    displays: { surveyId: string; createdAt: string }[];  // 问卷展示历史
    responses: string[];  // 已响应的问卷 ID 列表
    lastDisplayAt: string | null;  // 上次展示时间
    segments: string[];  // 用户所属分段
  };
  expiresAt: string | null;  // 用户状态过期时间
}
```

**过期机制**：
- 环境状态和用户状态都有 `expiresAt` 字段
- 初始化时检查是否过期，过期则重新从服务器同步
- 这样可以确保配置不会永久过期，同时减少不必要的网络请求

### 2.2 问卷进度层存储（IndexedDB）

**存储位置**：`packages/surveys/src/lib/offline-storage.ts`

**适用场景**：仅对 **Link 类型问卷** 且 **启用了 offlineSupport** 时生效

```typescript
// packages/surveys/src/components/general/survey.tsx:143-144
const offlinePersistEnabled =
  offlineSupport && isLinkSurvey && !isPreviewMode && !!appUrl && !!environmentId;
```

#### 2.2.1 IndexedDB 数据库结构

```typescript
// packages/surveys/src/lib/offline-storage.ts:8-12
const DB_NAME = "formbricks-offline";
const DB_VERSION = 1;

const STORE_PENDING_RESPONSES = "pendingResponses";  // 待发送响应
const STORE_SURVEY_PROGRESS = "surveyProgress";      // 问卷填写进度
```

#### 2.2.2 问卷进度对象结构

```typescript
// packages/surveys/src/lib/offline-storage.ts:32-42
export interface SurveyProgressEntry {
  surveyId: string;                    // 问卷 ID（主键）
  blockId: string;                     // 当前所在的块 ID
  responseData: TResponseData;         // 已填写的响应数据
  ttc: TResponseTtc;                   // 每题填写耗时
  currentVariables: TResponseVariables; // 当前变量值
  history: string[];                   // 浏览历史（用于返回上一题）
  selectedLanguage: string;            // 选择的语言
  surveyStateSnapshot: SerializedSurveyState;  // SurveyState 快照
  updatedAt: number;                   // 更新时间戳
}

export interface SerializedSurveyState {
  responseId: string | null;    // 响应 ID（如已创建）
  displayId: string | null;     // 展示 ID
  surveyId: string;
  singleUseId: string | null;   // 单次使用 ID（如适用）
  userId: string | null;
  contactId: string | null;
  responseAcc: TResponseUpdate; // 累积的响应数据
}
```

#### 2.2.3 待发送响应对象结构

```typescript
// packages/surveys/src/lib/offline-storage.ts:24-30
export interface PendingResponseEntry {
  id?: number;                        // 自增主键
  surveyId: string;                   // 问卷 ID（索引）
  responseUpdate: TResponseUpdate;    // 响应数据
  surveyStateSnapshot: SerializedSurveyState;  // 快照
  createdAt: number;                  // 创建时间戳
}
```

#### 2.2.4 IndexedDB 核心 API

```typescript
// packages/surveys/src/lib/offline-storage.ts:211-296

// 保存问卷进度
export const saveSurveyProgress = async (progress: SurveyProgressEntry): Promise<void> => {
  // 使用 put，同一 surveyId 会覆盖旧记录
  const request = store.put(progress);
};

// 读取问卷进度
export const getSurveyProgress = async (surveyId: string): Promise<SurveyProgressEntry | undefined> => {
  const request = store.get(surveyId);
};

// 部分更新问卷进度快照（如新增 responseId）
export const patchSurveyProgressSnapshot = async (
  surveyId: string,
  snapshotPatch: Partial<SerializedSurveyState>
): Promise<void> => {
  // 读取现有记录 → 合并 patch → 写回
};

// 清除问卷进度
export const clearSurveyProgress = async (surveyId: string): Promise<void> => {
  const request = store.delete(surveyId);
};

// 待发送响应相关
export const addPendingResponse = async (...): Promise<number> => { /* add 到 pendingResponses */ };
export const getPendingResponses = async (...): Promise<PendingResponseEntry[]> => { /* 按 surveyId 查询 */ };
export const removePendingResponse = async (id: number): Promise<void> => { /* 删除已发送 */ };
export const syncPersistedResponses = async (...): Promise<{ success: boolean; syncedCount: number }> => {
  // 按时间顺序发送所有待处理响应
  // 4xx 客户端错误（如 409 已完成）：删除并继续
  // 5xx 服务器错误：停止，下次重试
};
```

### 2.3 SurveyState 内存状态

`SurveyState` 类负责在内存中管理问卷会话状态：

```typescript
// packages/surveys/src/lib/survey-state.ts:3-124
export class SurveyState {
  responseId: string | null = null;           // 响应 ID（创建后获得）
  displayId: string | null = null;            // 展示 ID
  userId: string | null = null;
  contactId: string | null = null;
  surveyId: string;
  shouldCreateResponseFromState = false;      // 是否从累积状态创建响应
  responseAcc: TResponseUpdate = {            // 累积的响应数据
    finished: false,
    data: {},
    ttc: {},
    variables: {}
  };
  singleUseId: string | null;

  accumulateResponse(responseUpdate: TResponseUpdate) {
    // 合并新响应到累积数据
    this.responseAcc = {
      finished: responseUpdate.finished,
      ttc: { ...this.responseAcc.ttc, ...responseUpdate.ttc },
      data: { ...this.responseAcc.data, ...responseUpdate.data },
      variables: responseUpdate.variables ?? this.responseAcc.variables,
      // ... 其他字段
    };
  }

  clear() {
    this.responseId = null;
    this.shouldCreateResponseFromState = false;
    this.responseAcc = { finished: false, data: {}, ttc: {}, variables: {} };
  }
}
```

---

## 三、断点续填

断点续填功能在 **Link 问卷** 且启用 **offlineSupport** 时生效。核心流程分为：**进度保存**、**进度恢复**、**响应同步** 三个阶段。

### 3.1 进度保存

#### 3.1.1 触发时机

每次提交（点击下一步/完成）时保存进度：

```typescript
// packages/surveys/src/components/general/survey.tsx:919-1006
const onSubmit = async (surveyResponseData: TResponseData, responsettc: TResponseTtc) => {
  // ... 逻辑计算、状态更新 ...

  // --- Offline support: save progress on each submit ---
  if (offlinePersistEnabled) {
    const newBlockId = finished ? endingId || localSurvey.endings[0]?.id || "end" : nextBlockId || blockId;

    void saveSurveyProgress({
      surveyId: survey.id,
      blockId: newBlockId,
      responseData: { ...responseData, ...surveyResponseData },
      ttc: { ...ttc, ...responsettc },
      currentVariables: calculatedVariables,
      history: newHistory,
      selectedLanguage,
      surveyStateSnapshot: {
        responseId: surveyState?.responseId ?? null,
        displayId: surveyState?.displayId ?? null,
        surveyId: survey.id,
        singleUseId: surveyState?.singleUseId ?? null,
        userId: surveyState?.userId ?? null,
        contactId: surveyState?.contactId ?? null,
        responseAcc: surveyState?.responseAcc ?? { finished: false, data: {}, ttc: {}, variables: {} },
      },
      updatedAt: Date.now(),
    });
  }
};
```

#### 3.1.2 关键状态快照更新

当响应创建成功或展示创建成功时，需要更新快照中的 ID：

```typescript
// packages/surveys/src/components/general/survey.tsx:146-152
const persistSurveyStateSnapshot = useCallback(
  async (snapshotPatch: Partial<SerializedSurveyState>) => {
    if (!offlinePersistEnabled) return;
    await patchSurveyProgressSnapshot(survey.id, snapshotPatch);
  },
  [offlinePersistEnabled, survey.id]
);

// 创建响应后更新 responseId
// packages/surveys/src/components/general/survey.tsx:187-189
onResponseCreated: (responseId) => {
  void persistSurveyStateSnapshot({ responseId });
},

// 创建展示后更新 displayId
// packages/surveys/src/components/general/survey.tsx:349-351
surveyState.updateDisplayId(display.data.id);
responseQueue.updateSurveyState(surveyState);
await persistSurveyStateSnapshot({ displayId: display.data.id });
```

### 3.2 进度恢复

#### 3.2.1 恢复时机

问卷组件挂载时检查是否需要恢复：

```typescript
// packages/surveys/src/components/general/survey.tsx:267
const [progressRestored, setProgressRestored] = useState(!offlinePersistEnabled);
```

#### 3.2.2 恢复流程

```typescript
// packages/surveys/src/components/general/survey.tsx:433-543
useEffect(() => {
  if (!offlinePersistEnabled) return;

  let cancelled = false;

  const restore = async () => {
    // 1. 从 IndexedDB 读取进度
    const progress = await getSurveyProgress(survey.id);

    if (cancelled || !progress) {
      setProgressRestored(true);
      return;
    }

    // 2. 检查进度是否过期（超过 24 小时）
    const MAX_AGE_MS = 24 * 60 * 60 * 1000;
    if (Date.now() - progress.updatedAt > MAX_AGE_MS) {
      await clearSurveyProgress(survey.id);
      setProgressRestored(true);
      return;
    }

    // 3. 检查是否有待发送的响应
    const pendingCount = responseQueue ? await responseQueue.loadPersistedQueue() : 0;

    // 4. 如果问卷已完成且没有待发送响应，不恢复（直接开始新问卷）
    if (pendingCount === 0) {
      const isEndingCard = localSurvey.endings.some((e) => e.id === progress.blockId);
      const isResponseFinished = progress.surveyStateSnapshot?.responseAcc?.finished === true;

      if (isEndingCard || isResponseFinished) {
        await clearSurveyProgress(survey.id);
        setProgressRestored(true);
        return;
      }
    }

    if (pendingCount > 0) {
      setPendingSyncCount(pendingCount);
    }

    // 5. 验证保存的 blockId 是否仍然有效
    const blockExists =
      progress.blockId === "start" ||
      progress.blockId === "end" ||
      localSurvey.blocks.some((b) => b.id === progress.blockId) ||
      localSurvey.endings.some((e) => e.id === progress.blockId);

    if (blockExists) {
      // 6. 恢复 UI 状态
      setBlockId(progress.blockId);
      setResponseData(progress.responseData);
      setTtc(progress.ttc);
      setCurrentVariables(progress.currentVariables);
      setHistory(progress.history);
      setSelectedLanguage(progress.selectedLanguage);

      // 7. 恢复 SurveyState
      if (surveyState && progress.surveyStateSnapshot) {
        restoreSurveyStateFromSnapshot(surveyState, progress.surveyStateSnapshot, progress);
        responseQueue?.updateSurveyState(surveyState);
      }
    } else {
      // Block 不存在（问卷结构可能已改变）
      await clearSurveyProgress(survey.id);
      if (surveyState && progress.surveyStateSnapshot) {
        restoreSurveyStateFromSnapshot(surveyState, progress.surveyStateSnapshot, progress);
        responseQueue?.updateSurveyState(surveyState);
      }
    }

    setProgressRestored(true);
  };

  void restore();

  return () => { cancelled = true; };
}, []);
```

#### 3.2.3 SurveyState 恢复细节

```typescript
// packages/surveys/src/components/general/survey.tsx:42-64
const restoreSurveyStateFromSnapshot = (
  surveyState: SurveyState,
  snapshot: SerializedSurveyState,
  progress: { responseData: TResponseData; ttc: TResponseTtc; currentVariables: TResponseVariables }
): void => {
  if (snapshot.responseId) surveyState.updateResponseId(snapshot.responseId);
  if (snapshot.displayId) surveyState.updateDisplayId(snapshot.displayId);
  if (snapshot.userId) surveyState.updateUserId(snapshot.userId);
  if (snapshot.contactId) surveyState.updateContactId(snapshot.contactId);
  if (snapshot.singleUseId) surveyState.singleUseId = snapshot.singleUseId;
  surveyState.disableBootstrapResponseCreate();
  surveyState.responseAcc = {
    ...snapshot.responseAcc,
    data: progress.responseData,
    ttc: progress.ttc,
    variables: progress.currentVariables,
    displayId: snapshot.displayId ?? snapshot.responseAcc.displayId,
  };
};
```

#### 3.2.4 responseId 恢复的特殊处理

如果快照中有 `displayId` 但没有 `responseId`，尝试通过 displayId 查找 responseId：

```typescript
// packages/surveys/src/components/general/survey.tsx:494-518
if (pendingCount === 0 && !progress.surveyStateSnapshot.responseId) {
  if (progress.surveyStateSnapshot.displayId && apiClient) {
    const responseLookup = await apiClient.getResponseIdByDisplayId(
      progress.surveyStateSnapshot.displayId
    );

    if (responseLookup.ok && responseLookup.data.responseId) {
      surveyState.updateResponseId(responseLookup.data.responseId);
      await persistSurveyStateSnapshot({ responseId: responseLookup.data.responseId });
    } else if (responseLookup.ok) {
      // displayId 存在但没有对应的 response，需要从状态创建响应
      surveyState.enableBootstrapResponseCreate();
    } else if (responseLookup.error.status === 404) {
      // displayId 不存在，重置并创建新的
      surveyState.updateDisplayId(null);
      surveyState.enableBootstrapResponseCreate();
      await persistSurveyStateSnapshot({ displayId: null });
    }
  } else {
    surveyState.enableBootstrapResponseCreate();
  }
}
```

### 3.3 响应同步

#### 3.3.1 网络状态监听

```typescript
// packages/surveys/src/components/general/survey.tsx:550-598
useEffect(() => {
  if (!offlinePersistEnabled || !responseQueue || !progressRestored) return;

  // 离线时重置同步锁
  if (!isOnline) {
    isSyncingRef.current = false;
    return;
  }

  if (isSyncingRef.current) return;
  isSyncingRef.current = true;

  const syncPending = async () => {
    try {
      const count = await responseQueue.getPendingCount();
      if (count === 0) return;

      setIsSyncing(true);
      setPendingSyncCount(count);

      const result = await responseQueue.syncPersistedResponses((synced, total) => {
        setPendingSyncCount(total - synced);
      });

      setIsSyncing(false);
      setPendingSyncCount(0);

      if (result.success) {
        // 同步成功后清除问卷进度
        await clearSurveyProgress(survey.id);
        if (result.syncedCount > 0) {
          setIsResponseSendingFinished(true);
        }
      }
    } finally {
      isSyncingRef.current = false;
    }
  };

  void syncPending();
}, [isOnline, offlinePersistEnabled, responseQueue, progressRestored, survey.id]);
```

#### 3.3.2 ResponseQueue 同步机制

```typescript
// packages/surveys/src/lib/response-queue.ts:217-296
async syncPersistedResponses(
  onProgress?: (synced: number, total: number) => void
): Promise<{ success: boolean; syncedCount: number }> {
  // 并发锁，防止在线/离线切换时重复同步
  if (this.isSyncing) return { success: false, syncedCount: 0 };
  if (this.isRequestInProgress) return { success: false, syncedCount: 0 };
  
  this.isSyncing = true;

  try {
    const entries = await getPendingResponses(this.config.surveyId);
    if (entries.length === 0) return { success: true, syncedCount: 0 };

    const queueLengthBeforeSync = this.queue.length;
    let syncedCount = 0;

    for (const entry of entries) {
      // 仅在快照有 responseId 时恢复，否则让上一次发送的 responseId 继续使用
      if (entry.surveyStateSnapshot.responseId) {
        this.surveyState.updateResponseId(entry.surveyStateSnapshot.responseId);
      }

      let result = await this.sendResponse(entry.responseUpdate);

      // 如果 updateResponse 返回 404，说明 createResponse 可能未到达服务器
      // 重置 responseId 并重试作为新创建
      if (!result.ok && result.error?.status === 404 && this.surveyState.responseId !== null) {
        this.surveyState.responseId = null;
        if (entry.surveyStateSnapshot.displayId) {
          this.surveyState.updateDisplayId(entry.surveyStateSnapshot.displayId);
        }
        result = await this.sendResponse(entry.responseUpdate);
      }

      if (entry.id !== undefined) {
        if (result.ok) {
          await removePendingResponse(entry.id);  // 成功后删除
        } else if (result.error && result.error.status >= 400 && result.error.status < 500) {
          // 客户端错误（如 409 已完成）：删除并继续
          await removePendingResponse(entry.id);
          continue;
        } else {
          // 服务器错误：停止同步，下次重试
          return { success: false, syncedCount };
        }
      }

      syncedCount++;
      onProgress?.(syncedCount, entries.length);
    }

    // 只清理同步前已在队列中的项
    const removed = this.queue.splice(0, queueLengthBeforeSync);
    for (const item of removed) {
      this.pendingDbIds.delete(item);
    }

    // 同步期间新加入的项继续处理
    if (this.queue.length > 0) {
      this.processQueue();
    }

    return { success: true, syncedCount };
  } finally {
    this.isSyncing = false;
  }
}
```

### 3.4 页面离开提醒

为防止用户意外丢失进度，当有未完成的问卷或未发送的响应时，会在页面关闭前提醒：

```typescript
// packages/surveys/src/components/general/survey.tsx:600-620
useEffect(() => {
  const handleBeforeUnload = (e: BeforeUnloadEvent) => {
    // 已开始填写但未完成的问卷
    if (offlinePersistEnabled && history.length > 0 && !isSurveyFinished) {
      e.preventDefault();
      return;
    }
    // 有待发送的离线响应
    if (
      offlinePersistEnabled &&
      responseQueue &&
      (responseQueue.queue.length > 0 || pendingSyncCount > 0)
    ) {
      e.preventDefault();
    }
  };

  window.addEventListener("beforeunload", handleBeforeUnload);
  return () => window.removeEventListener("beforeunload", handleBeforeUnload);
}, [history.length, isSurveyFinished, offlinePersistEnabled, responseQueue, pendingSyncCount]);
```

### 3.5 进度清除时机

问卷进度在以下情况会被清除：

1. **进度过期**：超过 24 小时（`packages/surveys/src/components/general/survey.tsx:448-452`）
2. **问卷已完成且无待发送响应**：避免恢复到结束页（`packages/surveys/src/components/general/survey.tsx:460-468`）
3. **问卷结构变化**：保存的 blockId 不再存在（`packages/surveys/src/components/general/survey.tsx:523-532`）
4. **离线响应同步成功**：所有响应发送成功后（`packages/surveys/src/components/general/survey.tsx:585-590`）
5. **SurveyState.setSurveyId()**：切换问卷时自动清除（`packages/surveys/src/lib/survey-state.ts:31-34`）

---

## 四、数据流图

### 4.1 初始化数据流

```
用户调用 formbricks.setup()
        │
        ▼
┌─────────────────────────┐
│ 读取 localStorage       │
│ key: "formbricks-js"    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐     是     ┌───────────────────────┐
│ 已有有效配置？          │──────────▶│ 检查状态过期时间       │
└───────────┬─────────────┘            │ - environment.expiresAt│
            │ 否                        │ - user.expiresAt      │
            ▼                          └───────────┬───────────┘
┌─────────────────────────┐                       │ 过期
│ fetchEnvironmentState() │                       ▼
│ fetchUserState()        │            ┌───────────────────────┐
│ filterSurveys()         │            │ 重新从服务器同步       │
└───────────┬─────────────┘            └───────────┬───────────┘
            │                                     │
            ▼                                     ▼
┌─────────────────────────┐            ┌───────────────────────┐
│ Config.update()         │◄───────────│ 更新配置               │
│ → 写入 localStorage     │            └───────────────────────┘
└─────────────────────────┘
```

### 4.2 问卷填写数据流

```
用户回答问题 → onSubmit()
        │
        ▼
┌───────────────────────────────────┐
│ 1. 更新内存状态                    │
│    - responseData, ttc, variables │
│    - SurveyState.responseAcc      │
└───────────┬───────────────────────┘
            │
            ▼
┌───────────────────────────────────┐     是
│ offlinePersistEnabled?            │──────────┐
└───────────┬───────────────────────┘          │
            │ 否                                │
            ▼                                  ▼
┌──────────────────────────┐       ┌────────────────────────────┐
│ 直接发送到服务器          │       │ 2. saveSurveyProgress()   │
│ ResponseQueue.processQueue │     │    → IndexedDB            │
└──────────────────────────┘       └───────────┬────────────────┘
                                               │
                                               ▼
                                     ┌────────────────────────────┐
                                     │ 3. addPendingResponse()    │
                                     │    → IndexedDB              │
                                     └───────────┬────────────────┘
                                               │
                                               ▼
                                     ┌────────────────────────────┐
                                     │ 4. 尝试发送                 │
                                     │ 在线则立即发送               │
                                     │ 离线则暂存                   │
                                     └────────────────────────────┘
```

### 4.3 页面刷新恢复数据流

```
页面刷新 → Survey 组件挂载
        │
        ▼
┌────────────────────────────────┐
│ offlinePersistEnabled?         │────────┐
└───────────┬────────────────────┘        │
            │ 否                          │
            ▼                              ▼
┌────────────────────────┐      ┌─────────────────────────────┐
│ 从头开始新问卷          │      │ getSurveyProgress(surveyId) │
└────────────────────────┘      │ from IndexedDB              │
                                └───────────┬─────────────────┘
                                            │
                                            ▼
                                ┌─────────────────────────────┐
                                │ 检查进度有效性               │
                                │ - 未过期？（<24h）          │
                                │ - 问卷未完成？               │
                                │ - blockId 仍存在？          │
                                └───────────┬─────────────────┘
                                            │
                                            ▼
                                ┌─────────────────────────────┐
                                │ 恢复状态                     │
                                │ - blockId, responseData     │
                                │ - SurveyState 快照           │
                                │ - 待发送响应计数             │
                                └───────────┬─────────────────┘
                                            │
                                            ▼
                                ┌─────────────────────────────┐
                                │ 在线？                       │────────┐
                                └───────────┬─────────────────┘        │
                                            │ 否                        │ 是
                                            ▼                          ▼
                                ┌─────────────────────┐      ┌────────────────────────┐
                                │ 继续填写             │      │ syncPersistedResponses │
                                │ 进度保存在 IndexedDB │      │ 发送待处理响应          │
                                └─────────────────────┘      └───────────┬────────────┘
                                                                         │
                                                                         ▼
                                                               ┌────────────────────────┐
                                                               │ 发送成功？              │
                                                               └───────────┬────────────┘
                                                                           │
                                                                           ▼
                                                              ┌─────────────────────────┐
                                                              │ clearSurveyProgress()   │
                                                              │ 清除问卷进度            │
                                                              └─────────────────────────┘
```

---

## 五、关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `packages/js-core/src/index.ts` | SDK 对外 API 入口 |
| `packages/js-core/src/lib/common/setup.ts` | SDK 初始化核心流程 |
| `packages/js-core/src/lib/common/config.ts` | Config 单例 + localStorage 持久化 |
| `packages/js-core/src/lib/common/constants.ts` | 常量定义（JS_LOCAL_STORAGE_KEY 等） |
| `packages/surveys/src/lib/survey-state.ts` | SurveyState 内存状态管理 |
| `packages/surveys/src/lib/offline-storage.ts` | IndexedDB 问卷进度/离线响应存储 |
| `packages/surveys/src/lib/response-queue.ts` | 响应队列 + 离线同步机制 |
| `packages/surveys/src/components/general/survey.tsx` | 问卷组件（进度保存/恢复/同步主流程） |

---

## 六、总结

Formbricks SDK 的会话状态保留机制采用 **分层架构**：

1. **全局配置层**（localStorage）：存储 SDK 配置、环境状态、用户状态，支持过期机制和自动迁移
2. **问卷进度层**（IndexedDB）：存储 Link 问卷的填写进度和待发送响应，支持断点续填和离线同步
3. **内存状态层**（SurveyState）：管理当前问卷会话的内存状态，提供响应累积和状态快照能力

核心特性：
- 支持 **24 小时** 内的问卷进度保留
- 支持 **离线** 环境下的问卷填写和响应暂存
- 网络恢复后 **自动同步** 待发送响应
- 页面关闭前 **提醒** 用户防止意外丢失
- 问卷结构变化时 **智能降级**（保留响应数据，重置 UI 进度）
