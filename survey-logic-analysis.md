# 问卷跳转逻辑求值链路分析报告

## 1. 核心架构概述

### 1.1 Block 结构模型
问卷采用 **Block（块）** 结构，每个 Block 包含多个 Element（问题）。逻辑规则定义在 Block 级别，而非单个问题级别。

```typescript
TSurveyBlock {
  id: string;                // Block ID (CUID)
  name: string;              // 块名称
  elements: TSurveyElement[]; // 问题列表
  logic?: TSurveyBlockLogic[]; // 逻辑规则列表
  logicFallback?: string;    // 默认分支（条件都不满足时跳转的目标）
}
```

### 1.2 逻辑规则结构
每个 Block 可以包含多条逻辑规则，每条规则由 **条件组（ConditionGroup）** 和 **动作列表（Actions）** 组成。

```typescript
TSurveyBlockLogic {
  id: string;
  conditions: TConditionGroup;  // 条件组（支持嵌套）
  actions: TSurveyBlockLogicAction[];  // 动作列表
}
```

---

## 2. 条件求值链路详解

### 2.1 条件组结构
条件组支持 **递归嵌套**，通过连接器 `AND` / `OR` 组合多个条件或子条件组：

```typescript
TConditionGroup {
  id: string;
  connector: "and" | "or";
  conditions: (TSingleCondition | TConditionGroup)[];
}
```

### 2.2 单个条件结构
```typescript
TSingleCondition {
  id: string;
  leftOperand: TDynamicLogicFieldValue;  // 左操作数
  operator: TSurveyLogicConditionsOperator;  // 操作符 (38种)
  rightOperand?: TRightOperand;  // 右操作数（可选，取决于操作符）
}
```

**左/右操作数类型**：
- `element`：问卷问题答案
- `variable`：问卷变量值
- `hiddenField`：隐藏字段值
- `static`：静态值（仅右操作数支持）

### 2.3 求值流程（evaluateLogic）

**步骤 1：递归遍历条件组**
- 从最外层条件组开始，递归处理每个子条件/子条件组
- `AND` 连接器：所有条件为 true，结果才为 true（短路求值）
- `OR` 连接器：任一条件为 true，结果即为 true（短路求值）

**步骤 2：单个条件求值（evaluateSingleCondition）**
```
1. 获取左操作数值 → getLeftOperandValue()
2. 获取右操作数值（如需要）→ getRightOperandValue()
3. 根据 operator 类型执行比较
4. ✅ 关键：任何异常都返回 false（静默失败）
```

### 2.4 容易误判的求值细节

#### ❌ 静默失败陷阱
**代码位置**：`logic.ts:445-447`
```typescript
catch (e) {
  return false;
}
```
- 任何异常（不存在的问题ID、类型不匹配、无效操作符等）都**静默返回 false**
- 不会抛出错误或日志记录，问题难以排查
- **影响**：可能导致逻辑分支静默走入错误路径

#### 📦 数组相等比较的特殊处理
**代码位置**：`logic.ts:288-294`
```typescript
return (
  (Array.isArray(leftValue) &&
    leftValue.length === 1 &&
    typeof rightValue === "string" &&
    leftValue.includes(rightValue)) ||
  leftValue === rightValue
);
```
- 当左值是**长度为1的数组**，右值是**字符串**时，执行 `includes` 比较而非严格相等
- 可能不符合直觉：`["a"] == "a"` 结果为 `true`

#### 🎯 Matrix 问题的行级访问
**代码位置**：`logic.ts:148-168`
```typescript
if (leftOperand.meta && leftOperand.meta?.row !== undefined) {
  const rowIndex = Number(leftOperand.meta.row);
  // 通过行索引访问特定行的值
}
```
- Matrix 问题支持通过 `meta.row` 访问特定行，而非整个矩阵对象
- **边界情况**：行索引越界 → 返回 `undefined`，最终条件为 false

---

## 3. 动作执行流程（performActions）

### 3.1 三种动作类型
| 动作类型 | 目标 | 说明 |
|---------|------|------|
| `calculate` | 变量 | 对变量执行计算操作（add/subtract/multiply/divide/assign/concat） |
| `requireAnswer` | 问题 | 标记目标问题为必填 |
| `jumpToBlock` | Block | 跳转到指定 Block |

### 3.2 执行顺序与关键行为

**关键代码**：`logic.ts:64-79`
```typescript
actions.forEach((action) => {
  switch (action.objective) {
    case "calculate":
      // 执行计算并更新变量
      break;
    case "requireAnswer":
      requiredQuestionIds.push(action.target);
      break;
    case "jumpToBlock":
      if (!jumpTarget) {  // ⚠️ 只取第一个！
        jumpTarget = action.target;
      }
      break;
  }
});
```

### 3.3 容易误判的动作细节

#### ⚠️ 多个 jumpToBlock 时只取第一个
- 如果条件组满足后触发多个 `jumpToBlock` 动作，**只有第一个会生效**
- 后续的 jump 动作会被**静默忽略**，无任何警告
- **设计意图**：避免跳转冲突，但缺乏冲突提示

#### ➗ 除以零保护
**代码位置**：`blocks.ts:48-54` 和 `logic.ts:505-506`
- 定义时验证：Zod schema 会拒绝静态值为 0 的除法
- 运行时保护：如果运行时计算出现除以零，**保持原始变量值不变**

#### 📊 计算操作不回滚
- calculate 动作按顺序执行
- 如果后续动作失败，之前的变量修改**不会回滚**
- 但由于异常被捕获并返回 false，整个条件组被视为不满足时，calculate 的副作用是否保留需要查看调用方实现

---

## 4. 默认分支与边界情况（logicFallback）

### 4.1 默认分支机制
每个 Block 可以配置 `logicFallback` 字段，定义**所有逻辑规则都不满足时**的跳转目标。

```typescript
logicFallback?: ZSurveyBlockId;  // 条件都不满足时跳转的 Block ID
```

### 4.2 完整的跳转决策流程

```
                用户提交当前 Block
                       │
                       ▼
            ┌─────────────────────────┐
            │  遍历 Block.logic 列表  │
            └─────────────────────────┘
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
    有逻辑规则                  无逻辑规则
          │                         │
    ┌─────┴─────┐                   │
    │ 逐条求值  │                   │
    └─────┬─────┘                   │
          │                         │
  ┌───────┴───────┐                 │
  │ 有满足的规则？ │                 │
  └───────┬───────┘                 │
          │                         │
     是 ──┴── 否                     │
      │        │                     │
      ▼        ▼                     │
  执行动作   检查 logicFallback?     │
      │        │                     │
      │   是 ──┴── 否                │
      │   │          │               │
      │   ▼          ▼               │
      │ 跳转目标    按顺序到下一个块  │
      │   │          │               │
      └───┴──────────┴───────────────┘
                    │
                    ▼
            最终跳转目标确定
```

### 4.3 边界场景汇总

| 场景 | 行为 | 风险等级 |
|-----|------|---------|
| Block.logic 为空 | 按问卷定义顺序跳转到下一个 Block | 🟢 低 |
| 所有条件都不满足，无 logicFallback | 按顺序到下一个 Block | 🟢 低 |
| 所有条件都不满足，有 logicFallback | 跳转到 logicFallback 指定的块 | 🟡 中 |
| 条件求值发生异常 | 该条件返回 false，继续评估其他条件 | 🟠 高 |
| 满足多个逻辑规则 | 逐条执行所有满足的规则的动作 | 🟡 中 |
| 多个规则都触发 jumpToBlock | **所有规则中的第一个 jump 生效** | 🔴 严重 |
| jumpTarget 指向不存在的 Block | 由调用方处理（可能导致问卷卡死） | 🔴 严重 |

---

## 5. 操作符完整列表与注意事项

### 5.1 无需右操作数的操作符（12个）
| 操作符 | 适用场景 | 注意事项 |
|-------|---------|---------|
| `isSubmitted` | 文件上传、文本输入等 | 空字符串、null 判定为未提交 |
| `isSkipped` | 所有类型 | `""`, `null`, `undefined`, 空数组 判定为跳过 |
| `isClicked` / `isNotClicked` | CTA 按钮 | 值严格等于 "clicked" |
| `isAccepted` | 同意书 | 值严格等于 "accepted" |
| `isBooked` | 日历预约 | 非空且不为 "skipped" |
| `isPartiallySubmitted` | Matrix | 对象值中存在空字符串 |
| `isCompletelySubmitted` | Matrix | 对象值非空且无空字符串 |
| `isSet` / `isNotSet` | 隐藏字段、变量 | 非 null/undefined/空字符串 |
| `isEmpty` / `isNotEmpty` | Matrix 行、文本 | 严格等于空字符串 |

### 5.2 需要右操作数的操作符（26个）

**比较类**：`equals`, `doesNotEqual`, `isGreaterThan`, `isLessThan`, `isGreaterThanOrEqual`, `isLessThanOrEqual`

**字符串操作**：`contains`, `doesNotContain`, `startsWith`, `doesNotStartWith`, `endsWith`, `doesNotEndWith`

**数组包含**：`equalsOneOf`, `includesAllOf`, `includesOneOf`, `doesNotIncludeAllOf`, `doesNotIncludeOneOf`, `isAnyOf`

**日期比较**：`isAfter`, `isBefore`

---

## 6. 关键风险点总结

### 🚨 高风险问题

1. **异常静默吞噬**
   - 所有条件求值异常都返回 false，不暴露错误信息
   - 可能导致：配置错误的逻辑长期不被发现

2. **多个 jump 冲突无提示**
   - 多个规则触发 jumpToBlock 时，只取第一个
   - 无冲突警告，行为依赖规则定义顺序

3. **数组相等比较反直觉**
   - `["a"] == "a"` 被判定为 true
   - 可能导致多选问题的逻辑判断不符合预期

4. **jumpTarget 有效性无运行时验证**
   - 跳转到不存在的 Block ID 时无保护机制

### ⚠️ 中等风险问题

5. **变量计算的副作用**
   - 条件满足后执行 calculate 会修改变量值
   - 变量值变化可能影响后续条件的求值

6. **多语言下的选项比较**
   - MultipleChoice 选项比较使用本地化标签（label），而非 ID
   - 语言切换可能导致逻辑行为变化

7. **Matrix 行索引越界**
   - 行索引转换失败或超出范围时，静默返回 undefined
   - 最终条件返回 false，难以调试

---

## 7. 调试与验证建议

### 7.1 开发调试建议
1. 在 `evaluateLogic` 和 `performActions` 中添加临时调试日志
2. 特别关注异常捕获点，建议增加开发环境的错误提示
3. 添加 jump 冲突检测警告

### 7.2 逻辑配置最佳实践
1. 避免在同一 Block 中定义多个可能同时触发的 jumpToBlock 规则
2. 对于复杂逻辑，建议拆分为多个简单 Block 而非深度嵌套条件
3. 始终配置 `logicFallback` 作为安全网
4. 变量计算避免循环依赖（A 满足修改 B，B 满足又修改 A）
5. Matrix 问题的行索引从 0 开始，注意与 UI 显示的对应关系

---

## 8. 关键代码位置参考

| 功能模块 | 文件位置 | 关键行号 |
|---------|---------|---------|
| 条件组求值 | `packages/surveys/src/lib/logic.ts` | 28-48 |
| 动作执行 | `packages/surveys/src/lib/logic.ts` | 50-82 |
| 左操作数获取 | `packages/surveys/src/lib/logic.ts` | 84-181 |
| 单个条件求值 | `packages/surveys/src/lib/logic.ts` | 206-448 |
| Block 类型定义 | `packages/types/surveys/blocks.ts` | 124-151 |
| 逻辑类型定义 | `packages/types/surveys/logic.ts` | 1-246 |
