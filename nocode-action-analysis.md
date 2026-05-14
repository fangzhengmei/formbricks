# No-Code Action 监听器生命周期分析报告

## 1. 核心调用链还原

### 1.1 初始化路径：setup()

```
setup()  [packages/js-core/src/lib/common/setup.ts:65
    │
    ├─> addEventListeners()  [event-listeners.ts:20-27]
    │    ├─> addEnvironmentStateExpiryCheckListener()
    │    ├─> addUserStateExpiryCheckListener()
    │    ├─> addPageUrlEventListeners()     ──┐
    │    ├─> addClickEventListener()          │
    │    ├─> addExitIntentListener()          │ 5种业务
    │    └─> addScrollDepthListener()       ──┘
    │
    └─> addCleanupEventListeners()  [event-listeners.ts:29-41]
         └─> 注册 window.beforeunload 监听器
```

### 1.2 清理路径：removeAllEventListeners()

```
removeAllEventListeners()  [event-listeners.ts:57-66]
    │
    ├─> clearEnvironmentStateExpiryCheckListener()
    ├─> clearUserStateExpiryCheckListener()
    ├─> removePageUrlEventListeners()      ──┐
    ├─> removeClickEventListener()           │
    ├─> removeExitIntentListener()           │ 5种业务
    ├─> removeScrollDepthListener()        ──┘
    ├─> clearTimeOnPageTimers()
    └─> removeCleanupEventListeners()
```

### 1.3 tearDown() 实际行为

**关键发现**：`tearDown()` **不清理任何事件监听器！

```typescript
// setup.ts:328-345
export const tearDown = (): void => {
  // 仅重置 user state
  // 仅关闭当前打开的 survey
  // ❌ 不调用 removeAllEventListeners()
  // ❌ 不清理任何事件监听器
};
```

## 2. 各个监听器的问题分析

### 2.1 PageUrlEventListeners - 状态标记正确

```typescript
// no-code-action.ts:163-203
let arePageUrlEventListenersAdded = false;  // 模块级标记

// 添加时：检查标记 → 设置为 true
// 移除时：检查标记 → 设置为 false
```

**问题：✅ 正确防止重复添加
**问题**：无影响后续触发？——不影响，因为标记正确维护

### 2.2 ClickEventListener - 状态标记正确

```typescript
// no-code-action.ts:206
let isClickEventListenerAdded = false;
```

**问题**：✅ 正确防止重复添加

### 2.3 ExitIntentListener - 严重不一致

**添加逻辑**（no-code-action.ts:276-283）：
```typescript
export const addExitIntentListener = (): void => {
  if (typeof document !== "undefined" && !isExitIntentListenerAdded) {
    document
      .querySelector("body")
      ?.addEventListener("mouseleave", checkExitIntentWrapper as unknown as EventListener);
    isExitIntentListenerAdded = true;  // ← 即使 body 不存在也会设为 true!
  }
};
```

**移除逻辑**（no-code-action.ts:285-290）：
```typescript
export const removeExitIntentListener = (): void => {
  if (isExitIntentListenerAdded) {
    document.removeEventListener("mouseleave", checkExitIntentWrapper as unknown as EventListener);
    isExitIntentListenerAdded = false;
  }
};
```

**问题 1**：**body 不存在时的标志位行为

| 场景 | 实际监听器是否添加 | 标志位 isExitIntentListenerAdded |
|------|------------------|--------------------------------|
| body 存在 | ✅ 已添加 | true |
| body 不存在 | ❌ 未添加 | **true (错误)** |

**影响范围**：
- **正常生产场景**：当页面加载时 `body` 通常存在，极少出现问题
- **SSR/测试场景**：在 JSDOM 环境或页面初始化阶段，`body` 可能不存在
- **后续触发判断影响**：
  - ❌ 第一次 `addExitIntentListener()`：body 不存在 → 未添加监听器，但标记设为 true
  - ❌ 第二次 `addExitIntentListener()`：标记为 true → 直接返回，**永远无法添加监听器**
  - ⚠️ 结果：ExitIntent 功能在 body 延迟加载的页面上**完全失效**

---

**问题 2**：**监听器绑定目标不一致**

- **添加**：`document.querySelector("body")?.addEventListener(...)
- **移除**：`document.removeEventListener(...)

**影响范围**：
- 当 `body` 不存在时：remove 调用在 `document` 上，监听器根本没有在 `body` 上，无实质伤害
- 当 `body` 存在时：remove 仍在 `document` 上调用，**监听器永远不会被移除**（绑定在 `body` 上）
- ⚠️ **真实可达行为**：正常卸载后，mouseleave 事件仍会触发，造成内存泄漏

### 2.4 ScrollDepthListener - 状态标记正确

```typescript
// no-code-action.ts:293
let scrollDepthListenerAdded = false;
```

**问题**：✅ 正确防止重复添加

### 2.5 CleanupEventListeners - 严重设计缺陷

```typescript
// event-listeners.ts:29-55
let areRemoveEventListenersAdded = false;

export const addCleanupEventListeners = (): void => {
  if (areRemoveEventListenersAdded) return;
  window.addEventListener("beforeunload", () => {  // ← 匿名函数 A
    // 清理逻辑
  });
  areRemoveEventListenersAdded = true;
};

export const removeCleanupEventListeners = (): void => {
  if (!areRemoveEventListenersAdded) return;
  window.removeEventListener("beforeunload", () => {  // ← 匿名函数 B
    // 相同的清理逻辑（但函数引用不同！
  });
  areRemoveEventListenersAdded = false;
};
```

**核心问题**：**匿名函数引用不匹配

| 操作 | 添加的函数 | 移除的函数 | 实际移除效果 |
|------|----------|----------|-----------|
| 第一次 add | 匿名函数 A | - | - |
| remove | - | 匿名函数 B | ❌ **未移除**（引用不同） |
| 第二次 add | - | - | 由于标志位为 false，会**重复添加匿名函数 C |

**影响范围**：
- **真实可达行为**：每次 `setup()` → `removeAllEventListeners()` → 再次 `setup()` 循环都会**泄漏一个 beforeunload 监听器
- **页面刷新/关闭时**：清理逻辑会被**执行多次
- **重初始化/测试场景**：问题放大，内存泄漏累积

## 3. 真实可达行为 vs 仅测试场景问题分类

| 监听器类型 | 确定影响后续触发 | 仅重初始化/测试场景 | 问题描述 |
|------------|--------------|-------------------|---------|
| PageUrlEventListeners | ❌ 无 | ❌ 无 | 标记正确 |
| ClickEventListener | ❌ 无 | ❌ 无 | 标记正确 |
| ExitIntentListener | ✅ 是 | ✅ 是 | ① body 不存在时标记错误 → 永久失效<br>② 移除目标错误 → 监听器泄漏 |
| ScrollDepthListener | ❌ 无 | ❌ 无 | 标记正确 |
| CleanupEventListeners | ✅ 是 | ✅ 是 | 匿名函数引用不匹配 → 每次重初始化泄漏 |

## 4. 现有测试盲区分析

### 4.1 ExitIntentListener 测试盲区

**现有测试**（no-code-action.test.ts:340-391）：
- ✅ 测试了 `document` 为 undefined 的情况
- ✅ 测试了 `body` 为 null 时不调用 addEventListener
- ✅ 测试了 `clientY <= 0` 时不触发
- ❌ **未测试**：body 不存在后，后续能否重新添加
- ❌ **未测试**：remove 时的目标元素是否正确
- ❌ **未测试**：body 不存在后再次调用 add 是否能成功添加

**缺失的测试场景：
```typescript
// 场景：body 不存在后，后续 body 出现时能否正常工作
1. 模拟 body = null → 调用 addExitIntentListener()
2. 模拟 body 存在 → 再次调用 addExitIntentListener()
3. 验证：监听器应该被添加到 body 上

// 场景：remove 目标正确性验证
1. body 存在 → addExitIntentListener()
2. 调用 removeExitIntentListener()
3. 验证：body.removeEventListener 被调用（而非 document）
```

### 4.2 CleanupEventListeners 测试盲区

**现有测试**（event-listeners.test.ts）：
- ❌ **完全没有测试** addCleanupEventListeners / removeCleanupEventListeners
- ❌ **未测试**：重复 add → remove → add 循环的泄漏问题

**缺失的测试场景**：
```typescript
// 场景：监听器能否正确移除
1. 调用 addCleanupEventListeners()
2. 调用 removeCleanupEventListeners()
3. 触发 beforeunload 事件
4. 验证：清理逻辑不应被调用

// 场景：重初始化不泄漏
1. addCleanupEventListeners() → removeCleanupEventListeners()
2. addCleanupEventListeners() → removeCleanupEventListeners()
3. 触发 beforeunload 事件
4. 验证：清理逻辑只应执行 0 次（而非 2 次）
```

## 5. 结论修正：影响范围与严重程度

### 5.1 真实生产环境影响

| 问题 | 生产环境概率 | 严重程度 | 实际影响 |
|------|-----------|---------|---------|
| ExitIntent body 不存在标记错误 | **低** | **中** | SPA 页面加载阶段 body 不存在时，ExitIntent 永久失效 |
| ExitIntent 移除目标错误 | **高** | **中** | 监听器泄漏，页面生命周期内 mouseleave 可能多次触发 |
| Cleanup 匿名函数不匹配 | **中** | **低** | 页面刷新/关闭时清理逻辑执行多次 |

### 5.2 测试/重初始化场景影响

| 问题 | 影响 |
|------|------|
| ExitIntent body 不存在 | 测试环境 JSDOM 常见，标记错误导致测试不稳定 |
| Cleanup 匿名函数不匹配 | 重复 setup/teardown 循环导致内存泄漏累积 |

### 5.3 原始结论修正

**原始表述**（错误）**：所有监听器都有问题，影响所有后续触发判断

**修正表述**（正确）**：

> 只有 **ExitIntentListener** 和 **CleanupEventListeners** 存在真实问题：
> 1. **ExitIntentListener** 有两个问题：
>    - body 不存在时标志位错误设置导致 **确实会影响后续触发判断（永久失效）
>    - 移除目标错误导致 **确实会影响后续触发判断（监听器泄漏）**
> 2. **CleanupEventListeners** 匿名函数引用不匹配导致 **确实会影响后续触发（重初始化时泄漏）**
> 3. PageUrl、Click、ScrollDepth 监听器的状态标记正确，**不影响后续触发判断**

## 6. 修复建议

### 6.1 ExitIntentListener 修复

```typescript
export const addExitIntentListener = (): void => {
  if (typeof document !== "undefined" && !isExitIntentListenerAdded) {
    const body = document.querySelector("body");
    if (body) {
      body.addEventListener("mouseleave", checkExitIntentWrapper as unknown as EventListener);
      isExitIntentListenerAdded = true;  // 只有实际添加后才设为 true
    }
  }
};

export const removeExitIntentListener = (): void => {
  if (isExitIntentListenerAdded) {
    const body = document.querySelector("body");
    if (body) {
      body.removeEventListener("mouseleave", checkExitIntentWrapper as unknown as EventListener);
    }
    isExitIntentListenerAdded = false;
  }
};
```

### 6.2 CleanupEventListeners 修复

```typescript
const cleanupHandler = (): void => {
  clearEnvironmentStateExpiryCheckListener();
  clearUserStateExpiryCheckListener();
  removePageUrlEventListeners();
  removeClickEventListener();
  removeExitIntentListener();
  removeScrollDepthListener();
  clearTimeOnPageTimers();
};

export const addCleanupEventListeners = (): void => {
  if (areRemoveEventListenersAdded) return;
  window.addEventListener("beforeunload", cleanupHandler);  // 使用命名函数
  areRemoveEventListenersAdded = true;
};

export const removeCleanupEventListeners = (): void => {
  if (!areRemoveEventListenersAdded) return;
  window.removeEventListener("beforeunload", cleanupHandler);  // 相同引用
  areRemoveEventListenersAdded = false;
};
```

### 6.3 tearDown() 补充监听器清理

```typescript
export const tearDown = (): void => {
  // ... 现有逻辑
  removeAllEventListeners();  // ← 添加这行
  setIsSetup(false);
};
```
