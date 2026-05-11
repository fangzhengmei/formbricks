# 问卷触发机制分析报告

## 一、触发方式概览

Formbricks 的问卷触发系统采用**统一的 Action 模型**，三种触发方式最终都收敛到相同的触发判定和执行流程。

| 触发方式 | 类型 | 触发源 | 说明 |
|---------|------|--------|------|
| **页面访问触发** | noCode (pageView) | 浏览器 URL 变化 | 用户访问指定页面时触发 |
| **按钮点击触发** | noCode (click) | 鼠标点击事件 | 用户点击指定 DOM 元素时触发 |
| **代码触发** | code | SDK API 调用 | 开发者通过 `formbricks.track()` 手动触发 |

---

## 二、三类触发规则的定义形态

### 2.1 核心数据模型：ActionClass

所有触发方式都基于 `ActionClass` 模型（定义在 `packages/types/action-classes.ts:130-142`）：

```typescript
interface ActionClass {
  id: string;           // 唯一标识
  name: string;         // 动作名称（用于问卷触发器匹配）
  type: "code" | "noCode";  // 动作类型
  key: string | null;   // 仅 code 类型使用：代码标识符
  noCodeConfig: NoCodeConfig | null;  // 仅 noCode 类型使用：无代码配置
  environmentId: string;
}
```

### 2.2 页面访问触发（Page View）

**定义位置**: `packages/js-core/src/lib/survey/no-code-action.ts:122-157`

**配置形态**:
```typescript
{
  type: "pageView",
  urlFilters: [
    {
      value: "/checkout/success",  // URL 匹配值
      rule: "exactMatch"           // 匹配规则
    }
  ],
  urlFiltersConnector: "or" | "and"  // 多规则连接符
}
```

**URL 匹配规则**（`packages/js-core/src/lib/common/utils.ts:215-247`）:
- `exactMatch` - 精确匹配
- `contains` - 包含匹配
- `startsWith` - 前缀匹配
- `endsWith` - 后缀匹配
- `notMatch` - 不匹配
- `notContains` - 不包含
- `matchesRegex` - 正则匹配

### 2.3 按钮点击触发（Click）

**定义位置**: `packages/js-core/src/lib/survey/no-code-action.ts:208-243`

**配置形态**:
```typescript
{
  type: "click",
  elementSelector: {
    cssSelector: ".submit-btn",   // CSS 选择器
    innerHtml: "立即购买"           // 元素内容（二选一即可）
  },
  urlFilters: [...],              // 可选：限制在特定页面
  urlFiltersConnector: "or" | "and"
}
```

**点击匹配逻辑**（`packages/js-core/src/lib/common/utils.ts:307-347`）:
1. 优先检查点击目标是否直接匹配 CSS 选择器
2. 若不匹配，向上遍历 DOM 树寻找最近的匹配祖先（支持嵌套结构，如 `<svg>` 在 `<button>` 内）
3. 检查元素 innerHTML 是否匹配
4. 验证 URL 过滤条件

### 2.4 代码触发（Code）

**定义位置**: `packages/js-core/src/lib/survey/action.ts:50-73`

**配置形态**:
```typescript
{
  type: "code",
  key: "user-completed-purchase"  // 代码标识符
}
```

**调用方式**（SDK API, `packages/js-core/src/index.ts:78-80`）:
```javascript
// 开发者在代码中调用
formbricks.track("user-completed-purchase", {
  hiddenFields: { orderId: "12345" }  // 可选：隐藏字段
});
```

### 2.5 其他 NoCode 触发类型（扩展）

系统还支持更多 NoCode 触发类型：

| 类型 | 触发条件 | 配置 |
|------|---------|------|
| `exitIntent` | 用户鼠标移出视口顶部 | `urlFilters` |
| `fiftyPercentScroll` | 页面滚动超过 50% | `urlFilters` |
| `pageDwell` | 用户在页面停留指定秒数 | `timeInSeconds`, `urlFilters` |

---

## 三、判定顺序与触发流程

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     事件源层 (Event Source)                  │
├─────────────────┬─────────────────┬─────────────────────────┤
│  页面访问事件    │   点击事件       │  SDK track() 调用       │
│ (hashchange,   │ (click listener)│ (Action.trackCodeAction) │
│  pushstate...) │                 │                         │
└────────┬────────┴────────┬────────┴───────────┬─────────────┘
         │                 │                    │
         ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│               Action 追踪层 (Action Tracking)                │
│              no-code-action.ts / action.ts                  │
│  - 事件监听与分发                                            │
│  - URL 过滤匹配                                              │
│  - 元素选择器匹配                                            │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              问卷过滤层 (Survey Filtering)                   │
│                 packages/js-core/src/lib/common/utils.ts     │
│  1. displayOption 过滤                                       │
│  2. recontactDays 过滤                                       │
│  3. segment 过滤                                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              触发匹配层 (Trigger Matching)                   │
│                 packages/js-core/src/lib/survey/action.ts    │
│  - 匹配 survey.triggers 中的 actionClass.name               │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              渲染执行层 (Survey Rendering)                   │
│                 packages/js-core/src/lib/survey/widget.ts    │
│  - displayPercentage 概率判断                                │
│  - 语言检查                                                  │
│  - 延迟渲染 (delay)                                          │
│  - 加载并渲染 Survey 组件                                     │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 判定顺序详解

#### 第一步：问卷预过滤（Setup 时执行）

**函数**: `filterSurveys()` (`packages/js-core/src/lib/common/utils.ts:77-151`)

在 `setup()` 阶段，所有问卷会经过三层过滤：

1. **displayOption 过滤**
   - `respondMultiple` - 始终显示（可多次响应）
   - `displayOnce` - 仅当未显示过时显示
   - `displayMultiple` - 仅当未响应过时显示
   - `displaySome` - 显示次数 < displayLimit 且未响应

2. **recontactDays 过滤**
   - 检查上次显示距离现在的天数
   - 优先使用 survey.recontactDays，其次使用 project.recontactDays

3. **segment 过滤**
   - 无 userId：排除有 segment filters 的问卷
   - 有 userId：仅保留用户所属 segment 的问卷

#### 第二步：事件触发（运行时）

**页面访问触发流程** (`packages/js-core/src/lib/survey/no-code-action.ts:122-157`):

```
1. 监听历史事件: hashchange, popstate, pushstate, replacestate, load
2. 遍历 noCode actionClasses 中 type === "pageView" 的动作
3. 对每个动作:
   - 获取 urlFilters 和 connector (or/and)
   - 调用 handleUrlFilters() 检查当前 URL 是否匹配
   - 匹配成功 → 加入命令队列执行 trackNoCodeAction()
```

**点击触发流程** (`packages/js-core/src/lib/survey/no-code-action.ts:208-243`):

```
1. 全局监听 document.click 事件
2. 遍历 noCode actionClasses 中 type === "click" 的动作
3. 对每个动作:
   - 调用 evaluateNoCodeConfigClick() 检查点击目标
   - 检查 cssSelector 匹配（支持向上查找祖先）
   - 检查 innerHtml 匹配
   - 检查 urlFilters 匹配
   - 全部匹配 → 加入命令队列执行 trackNoCodeAction()
```

**代码触发流程** (`packages/js-core/src/lib/survey/action.ts:50-73`):

```
1. 调用 formbricks.track("code-identifier")
2. 在 actionClasses 中查找 type === "code" 且 key === 标识符的动作
3. 找到后 → 调用 trackAction(actionClass.name)
```

#### 第三步：Action 追踪（统一入口）

**函数**: `trackAction()` (`packages/js-core/src/lib/survey/action.ts:14-42`)

```typescript
export const trackAction = async (name, alias?, properties?) => {
  // 1. 获取经过预过滤的问卷列表
  const activeSurveys = appConfig.get().filteredSurveys;

  // 2. 遍历问卷，匹配触发器
  for (const survey of activeSurveys) {
    for (const trigger of survey.triggers) {
      // 3. 检查问卷触发器中的 actionClass.name 是否匹配
      if (trigger.actionClass.name === name) {
        // 4. 命中后触发问卷
        await triggerSurvey(survey, name, properties);
      }
    }
  }
};
```

#### 第四步：问卷触发与渲染

**函数**: `triggerSurvey()` (`packages/js-core/src/lib/survey/widget.ts:25-47`)

```typescript
export const triggerSurvey = async (survey, action?, properties?) => {
  // 1. 显示概率判断 (displayPercentage)
  if (survey.displayPercentage) {
    const shouldDisplay = shouldDisplayBasedOnPercentage(survey.displayPercentage);
    if (!shouldDisplay) return;  // 概率未命中，跳过
  }

  // 2. 处理隐藏字段
  const hiddenFieldsObject = handleHiddenFields(survey.hiddenFields, properties?.hiddenFields);

  // 3. 渲染问卷
  await renderWidget(survey, action, hiddenFieldsObject);
};
```

**函数**: `renderWidget()` (`packages/js-core/src/lib/survey/widget.ts:49-199`)

渲染前的检查：
1. **互斥检查**: `isSurveyRunning` - 同一时间只能显示一个问卷
2. **用户识别等待**: 等待待处理的用户识别完成（针对有 segment filters 的问卷）
3. **语言检查**: 多语言问卷需检查当前语言是否可用
4. **延迟渲染**: 应用 `survey.delay` 秒延迟
5. **渲染**: 调用 `formbricksSurveys.renderSurvey()`

---

## 四、命中后做什么

### 4.1 问卷渲染参数

触发命中后，调用 `renderSurvey()` 传入以下核心参数：

| 参数 | 来源 | 说明 |
|-----|------|------|
| `survey` | 匹配到的问卷对象 | 完整的问卷定义（问题、样式、逻辑等） |
| `action` | 触发的 action 名称 | 用于追踪是哪个动作触发的 |
| `hiddenFieldsRecord` | properties.hiddenFields | 用户传入的隐藏字段 |
| `languageCode` | 用户语言配置 | 当前显示语言 |
| `styling` | project/survey styling | 样式配置（支持问卷级覆盖） |
| `placement`, `overlay`, `clickOutside` | project/survey 配置 | 显示位置和行为 |

### 4.2 回调钩子

渲染时注册三个重要回调：

#### onDisplayCreated
```typescript
// packages/js-core/src/lib/survey/widget.ts:148-171
onDisplayCreated: () => {
  // 1. 记录显示记录
  const newDisplay = { surveyId: survey.id, createdAt: new Date() };
  // 2. 更新用户状态
  config.update({
    user: {
      ...previousConfig.user,
      data: {
        ...previousConfig.user.data,
        displays: [...existingDisplays, newDisplay],
        lastDisplayAt: new Date(),
      },
    },
    // 3. 重新过滤问卷（基于新的显示记录）
    filteredSurveys: filterSurveys(...),
  });
}
```

#### onResponseCreated
```typescript
// packages/js-core/src/lib/survey/widget.ts:172-190
onResponseCreated: () => {
  // 1. 记录响应记录
  const newResponses = [...responses, survey.id];
  // 2. 更新用户状态并重新过滤
  config.update({
    user: {
      ...config.get().user,
      data: {
        ...config.get().user.data,
        responses: newResponses,
      },
    },
    filteredSurveys: filterSurveys(...),
  });
}
```

#### onClose
```typescript
// packages/js-core/src/lib/survey/widget.ts:201-218
onClose: closeSurvey,
// closeSurvey 实现:
// 1. 从 DOM 移除问卷容器
// 2. 重新过滤问卷
// 3. setIsSurveyRunning(false) - 允许新问卷显示
```

### 4.3 命令队列机制

#### 4.3.1 CommandQueue 核心特性

`CommandQueue`（`packages/js-core/src/lib/common/command-queue.ts:25-119`）是一个**FIFO（先进先出）串行执行队列**，不是按类型优先级排序的优先级队列。

```typescript
export class CommandQueue {
  private queue: InternalQueueItem[] = [];  // 简单数组，按入队顺序消费
  private running = false;                  // 互斥锁，确保串行执行

  // 入队操作
  public add(command, type, shouldCheckSetupFlag, ...args) {
    this.queue.push(newItem);  // 追加到队尾
    if (!this.running) {
      void this.run();         // 队列为空时启动消费循环
    }
  }

  // 消费循环
  private async run() {
    this.running = true;
    while (this.queue.length > 0) {
      const currentItem = this.queue.shift();  // 从队首取出（FIFO）
      // ...执行命令
    }
    this.running = false;
  }
}
```

**关键特性**：
1. **严格串行**：`running` 标志确保同一时间只有一个命令在执行
2. **FIFO 消费**：按 `push()` 入队顺序 `shift()` 出队，无优先级排序
3. **批量消费**：`run()` 启动后会持续消费直到队列为空

#### 4.3.2 三种 CommandType 的实际区别

`CommandType` 枚举值**不用于排序**，只用于类型区分，触发不同的前置检查：

```typescript
enum CommandType {
  Setup,        // 0 — 不检查 setup 状态
  UserAction,   // 1 — 检查 setup 状态
  GeneralAction // 2 — 检查 setup 状态 + 等待 UpdateQueue
}
```

**执行时的差异**（`command-queue.ts:82-97`）：

| 类型 | checkSetup | 等待 UpdateQueue | 典型命令 |
|-----|-----------|-----------------|---------|
| `Setup` | false | 否 | `Setup.setup()` |
| `UserAction` | true | 否 | `User.setUserId()`, `Attribute.setAttributes()` |
| `GeneralAction` | true | **是** | `Action.trackCodeAction()`, `checkPageUrl()` |

**GeneralAction 的特殊处理**：
```typescript
if (currentItem.type === CommandType.GeneralAction) {
  // 执行前先等待用户属性更新完成（确保 segment 过滤准确）
  const updateQueue = UpdateQueue.getInstance();
  if (!updateQueue.isEmpty()) {
    await updateQueue.processUpdates();  // 阻塞等待
  }
}
```

#### 4.3.3 Setup 后的入队与执行顺序

典型场景：`setup()` 后立即调用 `setUserId()`

```typescript
// packages/js-core/src/index.ts:14-48
const setup = async (setupConfig) => {
  // 1. Setup 入队
  await queue.add(Setup.setup, CommandType.Setup, false, setupConfig);
  
  // 2. 等待 Setup 完成
  await queue.wait();
  
  // 3. 放入下一事件循环
  setTimeout(() => {
    void checkPageUrl();  // 内部会入队 GeneralAction
  }, 0);
};

// 开发者代码：
await formbricks.setup(config);
formbricks.setUserId("user-123");  // 入队 UserAction
```

**事件循环时序**：

```
┌──────────────────────────────────────────────────────────────┐
│                    当前事件循环（同步代码）                     │
├──────────────────────────────────────────────────────────────┤
│  1. queue.add(Setup.setup)                                   │
│     → queue: [Setup]                                         │
│     → run() 启动，开始消费                                    │
│                                                              │
│  2. await queue.wait()                                       │
│     → 阻塞，等待 run() 完成                                   │
│                                                              │
│  3. Setup 执行完成，queue: []                                 │
│     → queue.wait() 返回                                      │
│                                                              │
│  4. setTimeout(fn, 0)                                        │
│     → fn 被注册到宏任务队列，不立即执行                        │
│                                                              │
│  5. setup() 返回                                             │
│                                                              │
│  6. formbricks.setUserId("user-123")                         │
│     → queue.add(User.setUserId, UserAction)                  │
│     → queue: [UserAction]                                    │
│     → run() 启动，开始消费                                    │
│                                                              │
│  7. 当前事件循环结束                                           │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                    下一事件循环（宏任务）                       │
├──────────────────────────────────────────────────────────────┤
│  1. setTimeout 的回调执行                                     │
│     → checkPageUrl()                                         │
│     → queue.add(checkPageUrl, GeneralAction)                 │
│     → queue: [GeneralAction]                                 │
│     → run() 启动（或追加到正在运行的 run()）                    │
└──────────────────────────────────────────────────────────────┘
```

**实际执行顺序**：`Setup` → `UserAction(setUserId)` → `GeneralAction(checkPageUrl)`

#### 4.3.4 为什么 checkPageUrl 要放在下一事件循环

**代码注释说得很清楚**（`packages/js-core/src/index.ts:42-47`）：

```typescript
// Schedule checkPageUrl to run in the next event loop iteration.
// This ensures that any user actions (like setUserId) called synchronously after setup()
// will be queued BEFORE the page view actions are processed.
setTimeout(() => {
  void checkPageUrl();
}, 0);
```

**如果不放在下一事件循环会怎样？**

假设 `checkPageUrl()` 同步执行：

```typescript
const setup = async (setupConfig) => {
  await queue.add(Setup.setup, CommandType.Setup, false, setupConfig);
  await queue.wait();
  
  // ❌ 同步调用，立即入队
  checkPageUrl();  // queue: [GeneralAction]
};

// 开发者代码：
await formbricks.setup(config);   // setup() 内部 checkPageUrl 已入队
formbricks.setUserId("user-123"); // 后入队
```

此时 `checkPageUrl` 先于 `setUserId` 执行，导致：

1. **Segment 过滤不准确**：用户 ID 还未设置，无法正确判断用户所属 segment
2. **问卷可能被错误过滤**：有 segment filters 的问卷会被提前排除
3. **UserAction 等待链断裂**：GeneralAction 执行前会等待 UpdateQueue，但此时 setUserId 还没入队

**放在下一事件循环的好处**：

```
setup() 完成 → 返回 → setUserId() 入队（UserAction）
                ↓
           事件循环切换
                ↓
           setTimeout 回调执行 → checkPageUrl() 入队（GeneralAction）
```

这样 `setUserId` 始终比 `checkPageUrl` 先执行，确保：
- 用户身份先建立
- segment 过滤基于最新的用户状态
- GeneralAction 执行前 UpdateQueue 有机会处理完用户属性更新

---

## 五、关键文件索引

| 文件 | 职责 |
|-----|------|
| `packages/js-core/src/lib/survey/action.ts` | Action 追踪入口，问卷触发器匹配 |
| `packages/js-core/src/lib/survey/widget.ts` | 问卷渲染、显示概率、延迟、互斥控制 |
| `packages/js-core/src/lib/survey/no-code-action.ts` | NoCode 事件监听：页面访问、点击、退出意图、滚动、停留时间 |
| `packages/js-core/src/lib/common/utils.ts` | URL 匹配、问卷预过滤、点击元素匹配 |
| `packages/js-core/src/lib/common/setup.ts` | SDK 初始化、环境/用户状态同步 |
| `packages/js-core/src/index.ts` | SDK 公共 API 暴露 |
| `packages/types/action-classes.ts` | ActionClass 类型定义 |
| `packages/types/surveys/types.ts` | 问卷触发相关字段定义（triggers, displayOption 等） |

---

## 六、流程图总结

```
                    ┌─────────────────┐
                    │   初始化 Setup   │
                    │  filterSurveys  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  注册事件监听器   │
                    │ (页面/点击/滚动) │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  页面 URL 变化 │  │  用户点击元素  │  │ formbricks    │
│  (pageView)   │  │   (click)     │  │ .track(code)  │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
                  ┌────────────────┐
                  │  trackAction() │
                  │  匹配触发器名称  │
                  └───────┬────────┘
                          │
                          ▼
                  ┌────────────────┐
                  │ triggerSurvey()│
                  │ - 显示概率判断  │
                  │ - 隐藏字段处理  │
                  └───────┬────────┘
                          │
                          ▼
                  ┌────────────────┐
                  │ renderWidget() │
                  │ - 互斥检查      │
                  │ - 语言检查      │
                  │ - 延迟渲染      │
                  │ - 渲染问卷      │
                  └───────┬────────┘
                          │
                          ▼
                  ┌────────────────┐
                  │  问卷生命周期   │
                  │ displayCreated │
                  │ responseCreated│
                  │    onClose     │
                  └────────────────┘
```
