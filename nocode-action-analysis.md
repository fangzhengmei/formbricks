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

## 五、监听器生命周期与清理一致性分析（真实可达行为）

### 5.1 真实调用链还原

```
生产环境可达路径:
┌─────────────────────────────────────────────────────────────┐
│  setup()                                                    │
│    ├─ 首次调用: addEventListeners()  ← 添加所有监听器      │
│    └─ 首次调用: addCleanupEventListeners()  ← 添加 beforeunload │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  tearDown()  ← 只在 setUserId(不同用户) / logout 时调用     │
│    ├─ 重置用户状态为 DEFAULT_USER_STATE_NO_USER_ID          │
│    ├─ 更新 filteredSurveys                                   │
│    └─ closeSurvey()                                          │
│    ❗ 注意: 完全不调用任何 remove* 清理函数                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  removeAllEventListeners()                                   │
│    └─ 仅在测试代码中被调用，生产代码从未调用 ❗              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  beforeunload 事件                                           │
│    └─ 触发清理逻辑，但监听器本身永远无法被移除（见 5.3）     │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 注册与解绑对称性核对（修正版）

| 监听器类型 | 注册时机 | 生产环境是否会被解绑 | 对称状态 | 备注 |
|-----------|---------|---------------------|---------|------|
| **Page URL 事件** | setup | ❌ **永不** | ⚠️ 不对称 | History 补丁永久生效，无恢复机制 |
| **Click 事件** | setup | ❌ **永不** | ⚠️ 不对称 | 清理函数存在但从未被生产代码调用 |
| **Exit Intent 事件** | setup | ❌ **永不** | ❌ 严重不对称 | 注册目标: body，解绑目标: document，且解绑永不被调用 |
| **Scroll Depth 事件** | setup | ❌ **永不** | ⚠️ 不对称 | 存在时序问题，但实际永远不会触发解绑 |
| **PageDwell 定时器** | 页面 URL 变化时 | ✅ 页面切换时清理 | ✅ 相对对称 | 仅此类型有运行时清理 |
| **beforeunload 清理** | setup | ❌ **永不** | ❌ 严重 Bug | 匿名函数引用不匹配，永远无法移除 |

### 5.3 核心问题的真实影响范围

---

#### 🔴 问题 1: beforeunload 匿名函数不匹配（严重 Bug）

**代码位置**: `packages/js-core/src/lib/common/event-listeners.ts:29-55`

**问题本质**:
```typescript
window.addEventListener("beforeunload", () => { /* 匿名函数 A */ });
window.removeEventListener("beforeunload", () => { /* 匿名函数 B */ });
// ❌ () => {} !== () => {}，永远静默失败
```

**真实影响范围**:
- ✅ **不影响正常触发逻辑**：beforeunload 只在页面卸载时执行，不影响运行时的 action 触发
- ⚠️ **仅影响测试场景**：只有测试中才会调用 `removeCleanupEventListeners`
- ⚠️ **内存泄漏**：监听器永久残留，但页面卸载时内存会被回收，实际影响有限
- ❌ **不会导致重复触发**：beforeunload 只触发一次

**结论**: 这是一个代码质量问题，但**不影响用户可见的触发行为**，主要影响测试可靠性。

---

#### 🔴 问题 2: Exit Intent 注册/解绑目标不一致 + body 不存在时的标志位错误

**代码位置**: `packages/js-core/src/lib/survey/no-code-action.ts:276-290`

**问题本质 1**:
```typescript
document.querySelector("body")?.addEventListener("mouseleave", checkExitIntentWrapper);
// 注册在 body 上

document.removeEventListener("mouseleave", checkExitIntentWrapper);
// 解绑在 document 上
// ❌ 目标不同，永远无法移除
```

**问题本质 2（body 不存在时）**:
```typescript
export const addExitIntentListener = (): void => {
  if (typeof document !== "undefined" && !isExitIntentListenerAdded) {
    document.querySelector("body")?.addEventListener(...);
    // 👆 如果 body 不存在，可选链短路，不添加监听器
    isExitIntentListenerAdded = true;
    // 👆 但标志位仍被设为 true！
  }
};
```

**真实影响范围**:
- ✅ **正常浏览器环境**：body 存在时，监听器确实被添加到 body 上
- ❌ **但永远无法被移除**：即使调用解绑函数（实际不会被调用）也没用
- ❌ **SSR/iframe 沙箱/文档解析早期**：body 不存在时
  - 监听器**不被添加**
  - 但 `isExitIntentListenerAdded = true`
  - **后续即使 body 出现了，也永远无法再注册监听器**
- ⚠️ **对触发判断的影响**：Exit Intent action 可能在某些环境下完全无法触发

---

#### 🟠 问题 3: History API 补丁永久污染

**代码位置**: `packages/js-core/src/lib/survey/no-code-action.ts:167-188`

**问题本质**:
- 原始 `pushState`/`replaceState` 引用只存在于闭包中
- 没有任何 API 可以恢复原始方法
- `isHistoryPatched` 是模块级变量，一旦设为 true 就永远不会重置

**真实影响范围**:
- ✅ **直接影响 pageView 触发**：任何 pushState/replaceState 调用都会派发自定义事件
- ❌ **SPA 路由切换时重复触发风险**：即使 SDK 逻辑上"重置"了，patch 仍在
- ❌ **与其他库冲突风险**：如果有多个库 patch history，可能导致嵌套调用
- ⚠️ **仅重初始化/测试场景有影响**：正常单页应用中，SDK 只 setup 一次

---

#### 🟡 问题 4: Scroll Depth 时序问题

**问题本质**:
```typescript
if (document.readyState === "complete") {
  window.addEventListener("scroll", checkScrollDepthWrapper);
} else {
  window.addEventListener("load", () => {
    window.addEventListener("scroll", checkScrollDepthWrapper);
  });
}
scrollDepthListenerAdded = true;  // 立即设为 true，不管 scroll 监听器是否真的添加了
```

**真实影响范围**:
- ⚠️ **仅极端时序场景有影响**：在 `load` 事件触发前调用 remove（但生产中 remove 永不被调用）
- ✅ **正常生产流程无影响**：setup 通常在页面加载完成后执行，即使没完成，load 事件最终也会添加监听器

---

#### 🔴 问题 5: tearDown 完全不清理监听器（最大隐藏问题）

**代码位置**: `packages/js-core/src/lib/common/setup.ts:328-345`

**问题本质**:
```typescript
export const tearDown = (): void => {
  // ... 只重置用户状态和关闭调查
  closeSurvey();
  // ❗ 没有调用任何 remove*EventListeners 函数
  // ❗ 没有清理定时器
  // ❗ 没有重置任何标志位
};
```

**真实影响范围**:
- ❌ **切换用户后所有监听器仍在运行**：包括点击检测、退出意图检测、滚动检测
- ❌ **旧用户的 actionClasses 配置可能仍在生效**（取决于 Config 是否更新）
- ✅ **不影响新用户的触发逻辑**：新的 actionClasses 会覆盖旧的配置
- ⚠️ **仅多用户切换场景有影响**：单用户场景下永远不会遇到

---

### 5.4 测试盲区分析

| 测试文件 | 盲区位置 | 问题描述 | 实际后果 |
|---------|---------|---------|---------|
| `event-listeners.test.ts:99-104` | `removeCleanupEventListeners` 测试 | 用 `expect.any(Function)` 断言，这**永远通过** | 完全没检测到匿名函数不匹配的严重 Bug |
| `no-code-action.test.ts:276-286` | Exit Intent 解绑测试 | 分别 mock document 和 document.body，不验证目标一致性 | 测试通过，但实际代码永远无法解绑 |
| 所有测试 | | 没有测试"body 不存在"的边界场景 | 标志位错误无法被发现 |
| 所有测试 | | 没有测试"重复 setup/tearDown"场景 | 监听器残留问题无法被发现 |
| 所有测试 | | 只验证"函数被调用了"，不验证"监听器真的被移除了" | 所有解绑测试都是"假阳性" |

### 5.5 对触发判断的真实影响总结

| 问题 | 影响运行时触发吗？ | 仅测试场景？ | 严重程度 |
|-----|-------------------|------------|---------|
| beforeunload 匿名函数不匹配 | ❌ 不影响 | ✅ 仅测试 | 🟡 低 |
| Exit Intent 注册/解绑目标不一致 | ✅ 影响（SSR 环境） | ❌ | 🔴 高 |
| Exit Intent body 不存在时标志位错误 | ✅ 影响（SSR 环境） | ❌ | 🔴 高 |
| History 补丁永久污染 | ✅ 影响（重初始化场景） | ⚠️ 部分 | 🟠 中 |
| Scroll Depth 时序问题 | ❌ 几乎不影响 | ✅ 仅测试 | 🟡 低 |
| tearDown 不清理监听器 | ✅ 影响（多用户切换） | ❌ | 🟠 中 |

### 5.6 关键发现总结

1. **90% 的清理代码在生产环境中是死代码**：所有 `remove*` 函数除了在 beforeunload 中外，从未被实际调用
2. **最大的问题不是无法移除，而是移除函数本身就不会被调用**
3. **Exit Intent 有两个独立 Bug**：目标不一致 + body 不存在时标志位错误
4. **测试存在系统性盲区**：所有解绑测试都是"假阳性"，只验证函数被调用，不验证实际效果

---

## 六、关键代码位置

| 功能模块 | 文件路径 | 核心函数 |
|---------|---------|---------|
| NoCode 事件监听 | `packages/js-core/src/lib/survey/no-code-action.ts` | `addPageUrlEventListeners`, `addClickEventListener`, `addExitIntentListener`, `addScrollDepthListener`, `clearTimeOnPageTimers` |
| 事件清理管理 | `packages/js-core/src/lib/common/event-listeners.ts` | `addEventListeners`, `addCleanupEventListeners`, `removeCleanupEventListeners`, `removeAllEventListeners` |
| 匹配核心逻辑 | `packages/js-core/src/lib/common/utils.ts` | `handleUrlFilters`, `checkUrlMatch`, `evaluateNoCodeConfigClick` |
| Action 触发 | `packages/js-core/src/lib/survey/action.ts` | `trackNoCodeAction`, `trackAction` |
| 类型定义 | `packages/types/action-classes.ts` | `ZActionClassNoCodeConfig`, `TActionClassPageUrlRule` |
| Action 构建 | `apps/web/modules/survey/editor/lib/action-builder.ts` | `buildNoCodeAction`, `buildActionObject` |
| 表单验证 | `apps/web/modules/survey/editor/lib/action-utils.ts` | `createActionClassZodResolver` |

---

## 七、tearDown 跨用户触发风险专项分析

### 7.1 tearDown 实际执行内容（与预期对比）

| 预期应该做的 | 实际做了什么 | 缺失的影响 |
|------------|------------|----------|
| 清理所有监听器 | ❌ 完全不做 | 点击/滚动/退出意图监听器永久残留 |
| 清理 pageDwell 定时器 | ❌ 完全不做 | 旧用户的定时器仍会在未来触发 |
| 清理 TimeoutStack | ❌ 完全不做 | 超时调度残留 |
| 清理 CommandQueue | ❌ 完全不做 | 之前排队的命令仍会执行 |
| 重置 isSetup 标志 | ❌ 完全不做 | checkSetup 永远返回 ok |
| 重置用户状态为默认 | ✅ 做了 | 这是 tearDown 唯一做对的事情 |
| 重新计算 filteredSurveys | ✅ 做了 | 用 DEFAULT_USER_STATE_NO_USER_ID 过滤 |
| 关闭当前显示的调查 | ✅ 做了 | 调用 closeSurvey() |

### 7.2 跨用户触发时序推演（pageDwell 场景）

#### 场景设定
- 用户A 登录，在页面 `/dashboard` 停留
- pageDwell action: `timeOnPage_30s`，30秒后触发
- 用户A 有权看到 SurveyA（仅登录用户可见）
- 匿名用户（默认）无权看到 SurveyA

#### 精确时序
```
T0: 用户A 进入 /dashboard
    ↓
    checkPageUrl() 被调用
    ↓
    pageDwell 定时器启动（id = 123, actionName = "timeOnPage_30s"）
    ↓
    timeOnPageTimers.set("timeOnPage_30s", {
      status: "running",
      pageKey: "/dashboard",
      timerId: 123
    })

T0+10s: 调用 setUserId("用户B") / logout()
    ↓
    tearDown() 被调用
    ↓
    ✅ Config.user = DEFAULT_USER_STATE_NO_USER_ID（匿名用户）
    ✅ Config.filteredSurveys = filterSurveys(env, anonymousUser) → SurveyA 不在列表中
    ✅ closeSurvey() 被调用
    ↓
    ❗ 注意：timeOnPageTimers 中的定时器 123 完全没有被触碰！
    ❗ 注意：isSetup 标志仍然为 true！
    ❗ 注意：CommandQueue 完全没有被清理！

T0+29s: CommandQueue 中有用户A 之前触发的其他 action（如果有的话）
    ↓
    这些命令会正常执行，使用当前的（匿名用户）配置

T0+30s: 定时器 123 到期触发
    ↓
    timeOnPageTimers.set("timeOnPage_30s", { status: "fired", pageKey: "/dashboard" })
    ↓
    queue.add(trackNoCodeTimeOnPageActionHandler, CommandType.GeneralAction, true, "timeOnPage_30s")

T0+30s + 几毫秒: CommandQueue 执行该命令
    ↓
    checkSetup() → 返回 ok ✅（isSetup 仍为 true）
    ↓
    检查 UpdateQueue 为空，无需等待
    ↓
    trackAction("timeOnPage_30s") 被调用
    ↓
    appConfig.get().filteredSurveys → 使用当前的匿名用户配置
    ↓
    遍历 filteredSurveys，寻找 trigger.actionClass.name === "timeOnPage_30s"
    ↓
    最终结果取决于匿名用户是否有匹配的 survey trigger
```

### 7.3 真实影响边界分析

| 情况 | 是否会触发 | 归属用户 | 备注 |
|-----|-----------|---------|------|
| **匿名用户也有相同的 pageDwell action trigger** | ✅ 会触发 | 匿名用户 | ✅ 逻辑上"正确"，但时序来源是旧用户 |
| **只有登录用户有该 action trigger** | ❌ 不会触发 | 无 | ✅ 安全，因为 filteredSurveys 已更新为匿名用户 |
| **触发时还在排队等待 UpdateQueue** | ✅ 会等待新用户的更新完成 | 新用户 | ⚠️ 如果 setUserId 有后端请求，可能导致混淆 |
| **页面 URL 在定时器触发前改变了** | ❌ 不会触发 | 无 | ✅ checkTimeOnPage() 会因 URL 不匹配清理该定时器 |

### 7.4 可复现条件（三个条件必须同时满足）

1. **✅ 定时器启动后，页面 URL 没有发生变化**
   - 如果 URL 变化，checkTimeOnPage 会自动清理不匹配的定时器
   - SPA 单页应用在同一页面内切换用户最容易触发

2. **✅ tearDown 时 isSetup 标志仍为 true**
   - tearDown 不会改变 isSetup
   - 如果 tearDown 后又重新 setup，风险相同

3. **✅ 匿名用户/新用户的 filteredSurveys 包含该 pageDwell action trigger**
   - 最常见情况：所有用户（包括匿名）都有相同的 pageDwell action

### 7.5 不会造成跨用户行为归属的根本原因

**核心关键: trackAction 不携带用户身份上下文，只使用 Config 中的当前用户状态**

```typescript
export const trackAction = async (name: string): Promise<Result<void, NetworkError>> => {
  // 🔴 关键点：这里永远使用"当前"的 Config，不是启动定时器时的 Config
  const activeSurveys = appConfig.get().filteredSurveys;  // 快照当前用户配置
  
  for (const survey of activeSurveys) {
    for (const trigger of survey.triggers) {
      if (trigger.actionClass.name === name) {
        // 🔴 触发的是当前用户（匿名/新用户）的 survey，不是旧用户的
        await triggerSurvey(survey, name, properties);
      }
    }
  }
};
```

**结论**: 不会把旧用户的行为"归属"到新用户，因为：
- 新用户/匿名用户的 filteredSurveys 已经过滤掉了仅旧用户可见的 survey
- triggerSurvey 使用当前用户上下文创建响应

**但仍然有问题**: 这个 action 的触发时机来源是旧用户的行为，却在新用户身份下执行，可能造成数据分析时的时序混淆。

### 7.6 TimeoutStack 与 CommandQueue 的残留风险

#### TimeoutStack 残留
- TimeoutStack 存储的是 `{ event: string, timeoutId: number }`
- 仅用于调查显示超时（如自动关闭）
- tearDown 不清理，但 `closeSurvey()` 会取消当前显示的调查
- **风险很低**，因为超时 ID 与特定调查实例绑定

#### CommandQueue 残留
- CommandQueue 是 FIFO 队列
- tearDown 不清理队列
- 如果队列中有旧用户触发的 action（如 click action）
  - 会继续执行
  - 使用 tearDown 后的新用户配置（filteredSurveys）
  - **风险中等**，但符合"当前用户"语义

### 7.7 其他监听器的跨用户风险

| 监听器类型 | tearDown 后仍运行 | 风险等级 | 说明 |
|-----------|------------------|---------|------|
| **Click** | ✅ 是 | 🟡 低 | 点击事件是当前用户的真实行为，没有跨用户问题 |
| **Exit Intent** | ✅ 是 | 🟡 低 | 鼠标离开是当前用户的行为，没有跨用户问题 |
| **Scroll Depth** | ✅ 是 | 🟡 低 | 滚动是当前用户的行为，没有跨用户问题 |
| **Page View** | ✅ 是 | 🟠 中 | 如果新用户进入新页面，pageView 正常触发；问题是旧用户的 pageView 可能因为 History 补丁重复触发 |

### 7.8 风险总结与修复建议

#### 真实风险等级
| 风险 | 等级 | 说明 |
|-----|------|------|
| pageDwell 跨用户触发 | 🟡 低 | 仅时序混淆，不会造成错误归属 |
| CommandQueue 残留执行 | 🟡 低 | 符合当前用户语义 |
| History 补丁永久污染 | 🟠 中 | 可能导致重复 pageView 事件 |
| Exit Intent SSR 不注册 | 🔴 高 | 完全丢失退出意图事件 |

#### 修复建议（按优先级）
1. **🔴 高优先级**: 在 tearDown 中调用 `clearTimeOnPageTimers()` —— 一行代码解决主要隐患
2. **🔴 高优先级**: 修复 Exit Intent body 不存在时标志位错误
3. **🟠 中优先级**: 在 tearDown 中调用 `removeAllEventListeners()` 并在 setup 时重新注册
4. **🟡 低优先级**: 考虑清理 CommandQueue（但需要谨慎，可能中断正常流程）

---

## 八、设计特点总结（最终版）

### 优点
1. **模块化设计**: 每种事件类型独立管理，职责清晰
2. **容错性强**: 匹配失败静默处理，不影响主流程
3. **事件委托**: Click 事件支持祖先元素匹配，适配动态 DOM
4. **URL 功能丰富**: 7 种匹配规则 + AND/OR 连接器，满足复杂场景
5. **PageDwell 清理机制相对完善**: 页面切换时会主动清理不再匹配的定时器

### 真实影响生产的问题（按严重程度排序）

#### 🔴 高优先级（直接影响触发）
1. **Exit Intent body 不存在时标志位错误**: 在 SSR/iframe/文档解析早期环境下，Exit Intent 监听器永远无法注册，导致退出意图 action 完全不触发
2. **tearDown 不清理 pageDwell 定时器**: 多用户切换场景下，旧用户的定时器仍可能触发，造成数据分析时序混淆

#### 🟠 中优先级（边缘场景影响）
3. **tearDown 完全不清理监听器**: 多用户切换场景下监听器永久残留
4. **History 补丁永久污染**: 重初始化 SDK 时 patch 仍在，可能导致 pageView 重复触发
5. **Exit Intent 注册/解绑目标不一致**: 注册在 body，解绑在 document（虽然解绑永不被调用）

#### 🟡 低优先级（仅代码质量/测试问题）
6. **beforeunload 匿名函数不匹配**: 不影响生产功能，只影响测试可靠性
7. **Scroll Depth 时序问题**: 极端时序下才可能出现，生产中几乎不会遇到

### 测试与质量问题
1. **解绑测试系统性假阳性**: 所有 `remove*` 测试只验证函数被调用，不验证监听器真的被移除
2. **边界场景覆盖缺失**: 没有测试"body 不存在"、"重复 setup/tearDown"、"用户切换"等关键场景
3. **清理代码大部分是死代码**: 90% 的清理逻辑在生产中永远不会被执行

### 其他潜在优化点
1. **缺少匹配优先级**: 无法配置 action 优先级，完全依赖数组顺序
2. **缺少匹配统计**: 无法知道某个 action 匹配了多少次
3. **缺少调试钩子**: 匹配失败无日志，调试困难
4. **缺少 SPA 路由切换清理**: 单页应用页面切换时无自动清理机制
