# 问卷条件跳转（Logic Jump）实现机制分析报告

## 1. 存储结构

Formbricks 的条件跳转采用 **四层嵌套结构** 存储，数据以 JSON 格式存放在 Prisma 的 `Survey.blocks` 字段中。

### 1.1 数据库层

```prisma
// packages/database/schema.prisma:361
model Survey {
  // ...
  blocks  Json[]  @default([])  // 块数组，每个块包含逻辑配置
  endings Json[]  @default([])  // 结束卡，可作为跳转目标
  // ...
}
```

### 1.2 层级结构

```
Survey (问卷)
└── Block[] (块数组)
    ├── id: string                    // 块 ID (CUID)
    ├── name: string                  // 块名称
    ├── elements: SurveyElement[]     // 问题元素
    ├── logic: BlockLogic[]           // 逻辑规则数组 ← 核心
    └── logicFallback?: string        // 无匹配时的回退块 ID
```

### 1.3 BlockLogic 结构

每个块可以配置多条逻辑规则，每条规则定义了「条件 → 动作」的映射。

**位置**: `packages/types/surveys/blocks.ts:115-121`

```typescript
interface BlockLogic {
  id: string;
  conditions: ConditionGroup;  // 条件组（支持嵌套）
  actions: BlockLogicAction[]; // 动作数组
}
```

### 1.4 ConditionGroup 条件组

条件组支持 **嵌套** 和 **AND/OR** 连接符，形成灵活的条件表达式树。

**位置**: `packages/types/surveys/logic.ts:217-231`

```typescript
interface ConditionGroup {
  id: string;
  connector: "and" | "or";  // 连接符
  conditions: (SingleCondition | ConditionGroup)[];  // 支持嵌套
}
```

### 1.5 SingleCondition 单个条件

**位置**: `packages/types/surveys/logic.ts:138-175`

```typescript
interface SingleCondition {
  id: string;
  leftOperand: DynamicLogicFieldValue;  // 左操作数
  operator: LogicConditionsOperator;     // 比较操作符
  rightOperand?: RightOperand;          // 右操作数（部分操作符不需要）
}
```

#### 1.5.1 左操作数类型

左操作数支持三种数据源：

```typescript
// element: 问题/元素
{ type: "element", value: "questionId", meta?: { row: "0" } }

// variable: 变量
{ type: "variable", value: "variableId" }

// hiddenField: 隐藏字段
{ type: "hiddenField", value: "fieldId" }
```

#### 1.5.2 右操作数类型

```typescript
// static: 静态值
{ type: "static", value: "someValue" | 42 | ["opt1", "opt2"] }

// dynamic: 动态值（同左操作数）
{ type: "element", value: "anotherQuestionId" }
```

#### 1.5.3 支持的操作符

共 **38 种** 操作符，分为以下几类：

| 分类 | 操作符 | 说明 |
|------|--------|------|
| 相等 | `equals`, `doesNotEqual` | 等于/不等于 |
| 字符串 | `contains`, `doesNotContain`, `startsWith`, `doesNotStartWith`, `endsWith`, `doesNotEndWith` | 字符串包含/前缀/后缀 |
| 数值比较 | `isGreaterThan`, `isLessThan`, `isGreaterThanOrEqual`, `isLessThanOrEqual` | 数值大于/小于 |
| 日期比较 | `isAfter`, `isBefore` | 日期前后 |
| 数组包含 | `includesAllOf`, `includesOneOf`, `doesNotIncludeAllOf`, `doesNotIncludeOneOf` | 数组包含关系 |
| 状态检查 | `isSubmitted`, `isSkipped`, `isClicked`, `isNotClicked`, `isAccepted`, `isBooked`, `isPartiallySubmitted`, `isCompletelySubmitted` | 提交/跳过/点击状态 |
| 空值检查 | `isSet`, `isNotSet`, `isEmpty`, `isNotEmpty` | 是否设置/为空 |
| 多选 | `equalsOneOf`, `isAnyOf` | 等于多个值之一 |

**无需右操作数的操作符**：`isSubmitted`, `isSkipped`, `isClicked`, `isNotClicked`, `isAccepted`, `isBooked`, `isPartiallySubmitted`, `isCompletelySubmitted`, `isSet`, `isNotSet`, `isEmpty`, `isNotEmpty`

### 1.6 BlockLogicAction 动作

**位置**: `packages/types/surveys/blocks.ts:61-111`

动作分为三类：

#### 1.6.1 jumpToBlock（跳转）

```typescript
{
  id: string;
  objective: "jumpToBlock";
  target: string;  // 目标块 ID 或结束卡 ID
}
```

#### 1.6.2 requireAnswer（强制必填）

```typescript
{
  id: string;
  objective: "requireAnswer";
  target: string;  // 目标元素 ID
}
```

#### 1.6.3 calculate（变量计算）

```typescript
// 文本变量
{
  id: string;
  objective: "calculate";
  variableId: string;
  operator: "assign" | "concat";
  value: { type: "static", value: string } | DynamicLogicFieldValue;
}

// 数值变量
{
  id: string;
  objective: "calculate";
  variableId: string;
  operator: "add" | "subtract" | "multiply" | "divide" | "assign";
  value: { type: "static", value: number } | DynamicLogicFieldValue;
}
```

---

## 2. 运行时条件评估与跳转执行

### 2.1 入口函数

**位置**: `packages/surveys/src/components/general/survey.tsx:698-806`

`evaluateLogicAndGetNextBlockId` 是跳转逻辑的入口，在用户提交块时调用。

```typescript
const evaluateLogicAndGetNextBlockId = (
  data: TResponseData
): { nextBlockId: string | undefined; calculatedVariables: TResponseVariables } => {
  // 1. 合并当前响应数据
  const localResponseData = { ...responseData, ...data };
  let calculationResults = { ...currentVariables };

  // 2. 处理块级逻辑
  const evaluateBlockLogic = () => {
    let firstJumpTarget: string | undefined;
    const allRequiredQuestionIds: string[] = [];

    // 3. 遍历每条逻辑规则
    if (currentBlock.logic && currentBlock.logic.length > 0) {
      for (const logic of currentBlock.logic) {
        const result = processLogicRule(logic, firstJumpTarget, allRequiredQuestionIds);
        firstJumpTarget = result.jumpTarget;
        allRequiredQuestionIds.length = 0;
        allRequiredQuestionIds.push(...result.requiredIds);
        calculationResults = result.updatedCalculations;
      }
    }

    // 4. 使用 fallback（无匹配时）
    if (!firstJumpTarget && currentBlock.logicFallback) {
      firstJumpTarget = currentBlock.logicFallback;
    }

    return { firstJumpTarget, allRequiredQuestionIds };
  };

  // 5. 处理必填问题
  handleRequiredQuestions(allRequiredQuestionIds);

  // 6. 确定下一个块
  const nextBlockId = firstJumpTarget || localSurvey.blocks[currentBlockIndex + 1]?.id;
  return { nextBlockId, calculatedVariables: calculationResults };
};
```

### 2.2 评估顺序

**核心原则**：**按顺序评估，取第一个匹配的跳转目标**

```
流程：
┌─────────────────────────────────────────────────────────────┐
│ 1. 遍历 block.logic 数组（按配置顺序）                       │
│    ├── 对每条规则：                                          │
│    │   ├── 评估 conditions（递归遍历条件树）                 │
│    │   ├── 如果条件满足 → 执行 actions                        │
│    │   │   ├── 执行 calculate 动作（更新变量）               │
│    │   │   ├── 收集 requireAnswer 目标                       │
│    │   │   └── 收集 jumpToBlock 目标（只取第一个）           │
│    │   └── 如果条件不满足 → 跳过                              │
│    └── 继续下一条规则                                         │
│                                                              │
│ 2. 检查是否已有 jumpTarget                                    │
│    ├── 有 → 使用第一个 jumpTarget                            │
│    └── 无 → 检查 logicFallback                               │
│        ├── 有 → 使用 logicFallback                           │
│        └── 无 → 使用顺序下一个块                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 条件评估实现

**位置**: `packages/surveys/src/lib/logic.ts:28-48`

```typescript
export const evaluateLogic = (
  localSurvey, data, variablesData, conditions, selectedLanguage
): boolean => {
  const evaluateConditionGroup = (group: ConditionGroup): boolean => {
    // 递归评估所有子条件
    const results = group.conditions.map((condition) => {
      if (isConditionGroup(condition)) {
        return evaluateConditionGroup(condition);  // 递归
      } else {
        return evaluateSingleCondition(...);  // 评估单个条件
      }
    });

    // AND: 所有为真；OR: 任一为真
    return group.connector === "or" ? results.some(r => r) : results.every(r => r);
  };

  return evaluateConditionGroup(conditions);
};
```

### 2.4 动作执行实现

**位置**: `packages/surveys/src/lib/logic.ts:50-82`

```typescript
export const performActions = (survey, actions, data, calculationResults) => {
  let jumpTarget: string | undefined;
  const requiredQuestionIds: string[] = [];
  const calculations = { ...calculationResults };

  actions.forEach((action) => {
    switch (action.objective) {
      case "calculate":
        const result = performCalculation(...);
        if (result !== undefined) calculations[action.variableId] = result;
        break;
      case "requireAnswer":
        requiredQuestionIds.push(action.target);
        break;
      case "jumpToBlock":
        if (!jumpTarget) {  // 只取第一个
          jumpTarget = action.target;
        }
        break;
    }
  });

  return { jumpTarget, requiredQuestionIds, calculations };
};
```

**关键特性**：
- **多条跳转动作只取第一个**（`if (!jumpTarget)`）
- **变量计算按顺序累积**
- **强制必填问题可累积多个**

### 2.5 单个条件评估

**位置**: `packages/surveys/src/lib/logic.ts:206-448`

`evaluateSingleCondition` 函数处理 38 种操作符的具体逻辑。

#### 2.5.1 左操作数取值

根据类型从不同数据源获取值：

```typescript
const getLeftOperandValue = (localSurvey, data, variablesData, leftOperand, selectedLanguage) => {
  switch (leftOperand.type) {
    case "element":
      // 从响应数据中获取，支持：
      // - 数值类型转换
      // - 多选题的选项标签 → ID 映射
      // - 矩阵题的行索引取值
      return data[leftOperand.value];
    case "variable":
      return getVariableValue(variables, leftOperand.value, variablesData);
    case "hiddenField":
      return data[leftOperand.value];
  }
};
```

#### 2.5.2 特殊处理场景

| 场景 | 处理方式 |
|------|----------|
| 多选题比较 | 标签文本自动映射为选项 ID |
| 矩阵题 | 通过 `meta.row` 索引获取指定行的值 |
| 日期比较 | 使用 `Date.getTime()` 比较时间戳 |
| 异常处理 | 任何错误返回 `false`（条件不满足） |

---

## 3. 循环/死跳防护机制

### 3.1 循环检测算法

Formbricks 使用 **DFS（深度优先搜索）** 算法在 **验证阶段** 检测循环。

**位置**: `packages/types/surveys/blocks-validation.ts:3-81`

```typescript
export const findBlocksWithCyclicLogic = (blocks: TSurveyBlock[]): string[] => {
  const visited: Record<string, boolean> = {};      // 已访问标记
  const recStack: Record<string, boolean> = {};     // 递归栈（检测回边）
  const cyclicBlocks = new Set<string>();

  const checkForCyclicLogic = (blockId: string): boolean => {
    if (!visited[blockId]) {
      visited[blockId] = true;
      recStack[blockId] = true;

      const block = blocks.find((b) => b.id === blockId);

      // 路径1：检查 logic 中的 jumpToBlock 动作
      if (block?.logic && block.logic.length > 0) {
        for (const logic of block.logic) {
          const jumpActions = findJumpToBlockActions(logic.actions);
          for (const jumpAction of jumpActions) {
            const destination = jumpAction.target;

            // 跳过非块目标（结束卡）
            if (!blocks.find((b) => b.id === destination)) continue;

            if (!visited[destination] && checkForCyclicLogic(destination)) {
              cyclicBlocks.add(blockId);
              recStack[blockId] = false;
              return true;
            } else if (recStack[destination]) {
              // 发现回边 → 存在循环
              cyclicBlocks.add(blockId);
              recStack[blockId] = false;
              return true;
            }
          }
        }
      }

      // 路径2：检查 logicFallback
      if (block?.logicFallback) {
        const fallbackBlockId = block.logicFallback;
        if (blocks.find((b) => b.id === fallbackBlockId)) {
          // 同上，递归检测
        }
      }

      // 路径3：检查默认顺序（下一个块）
      const nextBlockIndex = blocks.findIndex((b) => b.id === blockId) + 1;
      const nextBlock = blocks[nextBlockIndex];
      if (nextBlock) {
        // 同上，递归检测
      }
    }

    recStack[blockId] = false;
    return false;
  };

  // 对每个块启动检测
  for (const block of blocks) {
    checkForCyclicLogic(block.id);
  }

  return Array.from(cyclicBlocks);
};
```

### 3.2 检测的三种路径

循环检测考虑 **所有可能的跳转路径**：

```
可能的跳转路径：
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Block A                                                     │
│  ├── logic[0].actions.jumpToBlock → Block X                  │
│  ├── logic[1].actions.jumpToBlock → Block Y                  │
│  ├── logicFallback → Block Z                                 │
│  └── 顺序下一个 → Block B                                     │
│                                                              │
│  所有这四条路径都需要检测是否形成循环                           │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 结束卡豁免

**结束卡（Ending Card）不会形成循环**，因此检测时会跳过：

```typescript
// 跳过非块目标（结束卡）
if (!blocks.find((b) => b.id === destination)) {
  continue;
}
```

### 3.4 验证时机

循环检测在 **问卷保存/验证阶段** 执行，而不是运行时。

**位置**: `packages/types/surveys/types.ts:3733-3746`

```typescript
// 验证逻辑循环
const logicIssues = validateBlockLogic(survey, blockIndex, block);
const logicFallbackIssue = validateBlockLogicFallback(survey, blockIndex, block);
const cyclicBlockIds = findBlocksWithCyclicLogic(survey.blocks);

// 发现循环则返回错误
if (cyclicBlockIds.includes(block.id)) {
  return {
    code: "custom",
    message: `Conditional Logic: Block ${String(blockIndex + 1)} forms a cycle`,
    path: ["blocks", blockIndex, "logic"],
  };
}
```

### 3.5 旧版问题级循环检测

**位置**: `packages/types/surveys/validation.ts:245-301`

与块级检测类似，旧版也有问题级的循环检测 `findQuestionsWithCyclicLogic`，逻辑相同。

---

## 4. 完整示例

### 4.1 存储示例

```json
{
  "blocks": [
    {
      "id": "block1",
      "name": "入门问题",
      "elements": [
        {
          "id": "q1",
          "type": "multipleChoiceSingle",
          "headline": { "default": "你是学生吗？" },
          "choices": [
            { "id": "yes", "label": { "default": "是" } },
            { "id": "no", "label": { "default": "不是" } }
          ]
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "equals",
                "rightOperand": { "type": "static", "value": "yes" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "block2"
            }
          ]
        }
      ],
      "logicFallback": "block3"
    },
    {
      "id": "block2",
      "name": "学生分支",
      "elements": [...]
    },
    {
      "id": "block3",
      "name": "非学生分支",
      "elements": [...]
    }
  ]
}
```

### 4.2 运行时流程

```
用户选择 "是" (q1 = "yes")
  ↓
评估 logic1.conditions
  ├── group1.connector = "and"
  └── cond1: q1 equals "yes" → true
  ↓
条件满足，执行 actions
  └── jumpToBlock → "block2"
  ↓
跳转到 block2（学生分支）


用户选择 "不是" (q1 = "no")
  ↓
评估 logic1.conditions
  └── cond1: q1 equals "yes" → false
  ↓
条件不满足，无 jumpTarget
  ↓
使用 logicFallback → "block3"
  ↓
跳转到 block3（非学生分支）
```

---

## 5. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 类型定义 | `packages/types/surveys/logic.ts` |
| 块类型定义 | `packages/types/surveys/blocks.ts` |
| 运行时评估 | `packages/surveys/src/lib/logic.ts` |
| 循环检测 | `packages/types/surveys/blocks-validation.ts` |
| 问卷组件（入口） | `packages/surveys/src/components/general/survey.tsx` |
| 数据库 Schema | `packages/database/schema.prisma` |
| 单元测试 | `packages/surveys/src/lib/logic.test.ts` |
