# NoCode Action 触发匹配规则分析报告

## 一、事件来源分析

NoCode Action 支持 **5 种** 不同的事件触发类型，每种类型都有独立的监听机制和触发条件：

### 1.1 事件类型概览

| 事件类型 | 触发条件 | 监听机制 |
|---------|---------|---------|
| **click** | 用户点击特定 DOM 元素 | 全局 click 事件监听 |
| **pageView** | 页面 URL 发生变化 | hashchange/popstate/pushstate/replacestate/load 事件 |
| **exitIntent** | 用户鼠标移出页面顶部 | mouseleave 事件监听 (body) |
| **fiftyPercentScroll** | 页面滚动超过 50% | scroll 事件监听 |
| **pageDwell** | 在页面停留指定时间 | setTimeout 定时器 |

### 1.2 各事件详细说明

#### Click 事件
- **触发源**: `document.addEventListener("click", ...)`
- **匹配元素**: 支持 CSS 选择器和 innerHTML 匹配
- **事件冒泡**: 自动向上查找祖先元素匹配（支持事件委托）
- **URL 过滤**: 支持 URL 过滤条件

#### PageView 事件
- **触发源**: URL 变化事件（hashchange/popstate）+ History API 补丁（pushstate/replacestate）
- **History 补丁**: 对 `history.pushState` 和 `history.replaceState` 进行 monkey patch，确保单页应用路由变化能被捕获
- **触发时机**: 页面加载完成（load 事件）和每次 URL 变化时

#### Exit Intent 事件
- **触发源**: `document.querySelector("body").addEventListener("mouseleave", ...)`
- **触发条件**: `e.clientY <= 0`（鼠标移出页面顶部）
- **URL 过滤**: 支持 URL 过滤条件

#### FiftyPercentScroll 事件
- **触发源**: `window.addEventListener("scroll", ...)`
- **触发条件**: `scrollPosition / (bodyHeight - windowSize) >= 0.5`
- **防重复**: `scrollDepthTriggered` 标志位，每页只触发一次
- **重置**: 滚动到顶部（`scrollPosition === 0`）时重置标志

#### PageDwell 事件
- **触发源**: setTimeout 定时器
- **状态管理**: `timeOnPageTimers` Map 存储每个 action 的定时器状态
- **页面切换处理**: 页面 URL 变化时会重启或清除相关定时器

---

## 二、匹配规则分析

### 2.1 URL 过滤规则（通用）

所有支持 URL 过滤的事件类型都使用统一的 `handleUrlFilters` 函数进行匹配。

#### 2.1.1 URL 匹配规则（7 种）

| 规则类型 | 匹配逻辑 |
|---------|---------|
| `exactMatch` | `url === pageUrlValue` |
| `contains` | `url.includes(pageUrlValue)` |
| `startsWith` | `url.startsWith(pageUrlValue)` |
| `endsWith` | `url.endsWith(pageUrlValue)` |
| `notMatch` | `url !== pageUrlValue` |
| `notContains` | `!url.includes(pageUrlValue)` |
| `matchesRegex` | `new RegExp(pageUrlValue).test(url)` |

> **注意**: 正则表达式匹配失败（无效 regex）时，返回 `false`（失败关闭策略）

#### 2.1.2 过滤器连接器

- **OR 连接器**（默认）: `urlFilters.some(...)` — 任一条件满足即匹配
- **AND 连接器**: `urlFilters.every(...)` — 所有条件都必须满足
- **空过滤器**: `urlFilters.length === 0` 时直接返回 `true`

### 2.2 Click 事件专属匹配

Click 事件在 `evaluateNoCodeConfigClick` 函数中实现，匹配流程如下：

```
1. 类型检查 → 确保是 click 类型
2. 必填检查 → cssSelector 和 innerHtml 至少一个存在
3. CSS 选择器匹配 →
   a. 先尝试直接匹配: targetElement.matches(cssSelector)
   b. 不匹配则向上查找祖先: targetElement.closest(cssSelector)
4. innerHTML 匹配 → 匹配元素的 innerHTML 必须完全相等
5. URL 过滤检查 → 应用通用 URL 过滤器
```

**关键点**:
- CSS 选择器匹配失败时会抛出异常被捕获，返回 `false`
- innerHTML 是**完全相等**匹配，不是包含匹配
- 匹配元素会被"解析"为实际触发元素（支持事件委托）

### 2.3 PageDwell 事件专属匹配

- 先通过 URL 过滤确定当前页面需要启动哪些定时器
- 定时器按 action name 管理，相同 action 在同一页面不会重复启动
- 页面 URL 变化时：
  - 不再匹配的 action：清除定时器并删除
  - 仍匹配但页面变更：重启定时器

---

## 三、匹配顺序与优先级

### 3.1 事件监听注册顺序

SDK 初始化时按以下顺序注册监听器（隐含优先级）：

1. Page URL 事件监听器（PageView + PageDwell）
2. Click 事件监听器
3. Exit Intent 事件监听器
4. Scroll Depth 事件监听器

### 3.2 Action 遍历顺序

**所有事件类型都遵循相同的遍历规则**：

```javascript
// 伪代码
for (const action of actionClasses) {
  if (action.type === "noCode" && action.noCodeConfig?.type === "[目标类型]") {
    // 进行匹配检查
    if (匹配成功) {
      // 触发 action
    }
  }
}
```

**关键特性**:

1. **顺序遍历**: 按 `actionClasses` 数组的顺序逐个检查
2. **全部匹配**: 多个 action 匹配成功时**全部都会触发**，没有"短路"逻辑
3. **无优先级**: 没有优先级权重配置，完全依赖数组顺序
4. **独立触发**: 每个匹配的 action 都会独立加入命令队列执行

### 3.3 命令队列执行

- 匹配成功的 action 通过 `CommandQueue.getInstance().add(...)` 加入队列
- 队列保证按加入顺序执行
- 支持去重参数（`CommandType.GeneralAction` + `true`）

---

## 四、未命中处理机制

### 4.1 通用未命中行为

**所有匹配失败的情况都采取"静默失败"策略**：

- ✅ 不抛出异常
- ✅ 不记录错误日志（仅 debug 级别日志）
- ✅ 不触发任何回调
- ✅ 直接跳过继续处理下一个 action

### 4.2 各事件未命中细节

#### Click 事件
- 匹配失败：直接 `continue` 循环，无任何副作用
- CSS 选择器无效：捕获异常并返回 `false`

#### PageView 事件
- URL 不匹配：检查是否有已调度的超时任务
  - 如果有：移除超时任务并设置 `isSurveyRunning = false`
  - 如果没有：无操作

#### Exit Intent 事件
- 鼠标未从顶部离开：不进入循环体
- URL 不匹配：`continue` 跳过

#### Scroll Depth 事件
- 未达到 50% 滚动深度：不进入匹配逻辑
- URL 不匹配：`continue` 跳过

#### PageDwell 事件
- URL 不匹配：
  - 运行中定时器：清除并删除
  - 已触发定时器：仅删除记录
  - 同时清理 TimeoutStack 中相关调度

### 4.3 错误边界处理

**仅在 trackNoCodeXxxActionHandler 中进行错误捕获**：

```javascript
// 工厂函数创建带上下文的处理器
const trackNoCodeClickActionHandler = createTrackNoCodeActionWithContext("click");

// 内部捕获并记录错误
if (!result.ok) {
  console.error(`🧱 Formbricks - Error in no-code ${context} action '${actionName}': ...`);
}
```

**注意**: 匹配阶段的错误不会走到这里，只有 action 触发后的执行错误才会被捕获。

---

## 五、监听器生命周期与清理一致性分析

### 5.1 注册与解绑对称性核对

| 监听器类型 | 注册函数 | 解绑函数 | 对称状态 | 备注 |
|-----------|---------|---------|---------|------|
| **Page URL 事件** | `addPageUrlEventListeners` | `removePageUrlEventListeners` | ✅ 基本对称 | 但 History 补丁无法恢复 |
| **Click 事件** | `addClickEventListener` | `removeClickEventListener` | ✅ 对称 | 目标: `document` |
| **Exit Intent 事件** | `addExitIntentListener` | `removeExitIntentListener` | ⚠️ 不对称 | 注册目标: `document.body`, 解绑目标: `document` |
| **Scroll Depth 事件** | `addScrollDepthListener` | `removeScrollDepthListener` | ⚠️ 潜在不对称 | `load` 事件触发后才真正注册 |
| **PageDwell 定时器** | (内部创建) | `clearTimeOnPageTimers` | ✅ 对称 | 独立函数清理 |
| **beforeunload 清理** | `addCleanupEventListeners` | `removeCleanupEventListeners` | ❌ **严重 Bug** | 函数引用不匹配 |

### 5.2 History API 补丁的永久影响

**问题描述**:
- `addPageUrlEventListeners` 对 `history.pushState` 和 `history.replaceState` 进行 monkey patch
- **补丁一旦应用就永远无法恢复**，因为原始函数引用只存在于函数闭包中
- 没有提供任何恢复原始 history 方法的 API

**代码位置** (`packages/js-core/src/lib/survey/no-code-action.ts:167-188`):
```typescript
if (!isHistoryPatched) {
  const originalPushState = history.pushState;
  history.pushState = function (...args) {
    originalPushState.apply(this, args);
    const event = new Event("pushstate");
    window.dispatchEvent(event);
  };
  // ... replaceState 同样的处理
  isHistoryPatched = true;
}
```

**潜在影响**:
1. **内存泄漏**: 闭包持有原始函数引用，无法被 GC
2. **行为污染**: 即使 SDK 被"卸载"，history 方法仍然会派发额外事件
3. **兼容性风险**: 可能与其他也 patch history 的库冲突
4. **多次 patch 风险**: 如果 `isHistoryPatched` 标志被意外重置，可能导致嵌套 patch

### 5.3 Exit Intent 监听器目标不一致

**问题代码**:
```typescript
// 注册: no-code-action.ts:278-280
document.querySelector("body")?.addEventListener("mouseleave", checkExitIntentWrapper);

// 解绑: no-code-action.ts:286-288
document.removeEventListener("mouseleave", checkExitIntentWrapper);
```

**问题描述**:
- 注册时目标是 **`document.body`**
- 解绑时目标是 **`document`**
- 根据 DOM 事件规范，事件目标不同时，`removeEventListener` 会静默失败

**实际影响**:
- ❌ Exit Intent 监听器**永远无法被正确移除**
- ❌ 即使调用 `removeExitIntentListener`，监听器仍然存在
- ❌ 页面跳转后可能残留，导致重复触发或内存泄漏

### 5.4 beforeunload 清理监听器的严重 Bug

**问题代码** (`packages/js-core/src/lib/common/event-listeners.ts:29-55`):
```typescript
export const addCleanupEventListeners = (): void => {
  if (areRemoveEventListenersAdded) return;
  window.addEventListener("beforeunload", () => {  // 匿名函数 A
    // ... 清理逻辑
  });
  areRemoveEventListenersAdded = true;
};

export const removeCleanupEventListeners = (): void => {
  if (!areRemoveEventListenersAdded) return;
  window.removeEventListener("beforeunload", () => {  // 匿名函数 B - 不相等!
    // ... 相同的清理逻辑
  });
  areRemoveEventListenersAdded = false;
};
```

**Bug 核心原因**:
- `addEventListener` 和 `removeEventListener` 使用了**两个不同的匿名函数实例**
- 在 JavaScript 中，`() => {} !== () => {}`，即使函数体完全相同
- 因此 `removeEventListener` **静默失败**，监听器永远不会被移除

**实际影响**:
1. beforeunload 监听器永久残留
2. 页面刷新/关闭时清理逻辑可能被多次执行
3. 内存泄漏（闭包持有整个 SDK 状态引用）
4. 标志位 `areRemoveEventListenersAdded` 与实际状态不一致

### 5.5 Scroll Depth 监听器的时序问题

**问题场景**:
```typescript
export const addScrollDepthListener = (): void => {
  if (document.readyState === "complete") {
    window.addEventListener("scroll", checkScrollDepthWrapper);
  } else {
    window.addEventListener("load", () => {
      window.addEventListener("scroll", checkScrollDepthWrapper);
    });
  }
  scrollDepthListenerAdded = true;
};
```

**潜在问题**:
1. 如果在 `load` 事件触发前调用 `removeScrollDepthListener`:
   - 标志位设为 `false`
   - 但 `load` 事件仍会在未来触发，最终还是会添加 scroll 监听器
   - 导致"已移除"但实际仍在监听的不一致状态

2. 没有取消 `load` 事件监听器的机制

### 5.6 监听器残留对触发判断的影响

| 残留监听器 | 对触发判断的影响 | 严重程度 |
|-----------|----------------|---------|
| **History 补丁残留** | pushState/replaceState 仍会派发自定义事件 → pageView 逻辑可能被意外触发 | 🔴 高 |
| **Exit Intent 残留** | mouseleave 仍会检测 → 页面卸载后仍可能触发 exitIntent action | 🟠 中 |
| **beforeunload 清理残留** | 页面卸载时清理逻辑重复执行 → 可能导致状态异常 | 🟡 低 |
| **Scroll Depth 时序残留** | scroll 事件仍会检测滚动深度 → 页面切换后仍可能触发 | 🟠 中 |
| **Click 监听器残留** | 点击仍会匹配元素 → 非预期页面也可能触发 click action | 🟠 中 |

### 5.7 页面生命周期中的清理时机

| 清理时机 | 实际行为 | 问题 |
|---------|---------|------|
| **beforeunload 事件** | 调用所有移除函数 + clearTimeOnPageTimers | ✅ 意图正确，但实现有 Bug |
| **手动调用 removeAllEventListeners** | 调用所有移除函数 | ❌ History 补丁不会恢复 |
| **SPA 页面切换** | 无自动清理 | ❌ 监听器全部残留 |
| **热更新/HMR** | 无特殊处理 | ❌ 可能导致重复注册 |

---

## 六、关键代码位置

| 功能模块 | 文件路径 | 核心函数 |
|---------|---------|---------|
| NoCode 事件监听 | `packages/js-core/src/lib/survey/no-code-action.ts` | `addPageUrlEventListeners`, `addClickEventListener`, `addExitIntentListener`, `addScrollDepthListener` |
| 匹配核心逻辑 | `packages/js-core/src/lib/common/utils.ts` | `handleUrlFilters`, `checkUrlMatch`, `evaluateNoCodeConfigClick` |
| Action 触发 | `packages/js-core/src/lib/survey/action.ts` | `trackNoCodeAction`, `trackAction` |
| 类型定义 | `packages/types/action-classes.ts` | `ZActionClassNoCodeConfig`, `TActionClassPageUrlRule` |
| Action 构建 | `apps/web/modules/survey/editor/lib/action-builder.ts` | `buildNoCodeAction`, `buildActionObject` |
| 表单验证 | `apps/web/modules/survey/editor/lib/action-utils.ts` | `createActionClassZodResolver` |

---

## 六、设计特点总结

### 优点
1. **模块化设计**: 每种事件类型独立管理，职责清晰
2. **容错性强**: 匹配失败静默处理，不影响主流程
3. **事件委托**: Click 事件支持祖先元素匹配，适配动态 DOM
4. **URL 功能丰富**: 7 种匹配规则 + AND/OR 连接器，满足复杂场景

### 潜在优化点
1. **缺少匹配优先级**: 无法配置 action 优先级，完全依赖数组顺序
2. **缺少匹配统计**: 无法知道某个 action 匹配了多少次
3. **缺少调试钩子**: 匹配失败无日志，调试困难
4. **正则无超时**: 恶意正则可能导致性能问题（虽然概率低）
