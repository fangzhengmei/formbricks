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

### 2.1 条件组结构与求值过程
条件组支持 **递归嵌套**，通过连接器 `AND` / `OR` 组合多个条件或子条件组：

```typescript
TConditionGroup {
  id: string;
  connector: "and" | "or";
  conditions: (TSingleCondition | TConditionGroup)[];
}
```

**⚠️ 重要：条件组求值过程不是短路执行**（代码位置：`packages/surveys/src/lib/logic.ts:35-45`）

```typescript
const evaluateConditionGroup = (group: TConditionGroup): boolean => {
  // 第一步：先用 map 逐项计算所有条件结果，全部执行完毕
  const results = group.conditions.map((condition) => {
    if (isConditionGroup(condition)) {
      return evaluateConditionGroup(condition);  // 递归处理嵌套组
    } else {
      return evaluateSingleCondition(...);        // 计算单个条件
    }
  });

  // 第二步：用 every/some 汇总所有结果
  return group.connector === "or" ? results.some((r) => r) : results.every((r) => r);
};
```

**真实执行顺序**：
1. **全部先算**：`group.conditions.map()` 会遍历并执行**所有**条件/子组，收集全部结果到 `results` 数组
2. **再汇总**：最后才用 `every()`（AND）或 `some()`（OR）判断最终结果
3. **无短路**：即使第一个条件已经能决定结果（如 AND 的第一个 false，OR 的第一个 true），后续条件仍会全部执行
4. **嵌套组亦然**：嵌套的条件组同样遵循「先算完所有子项，再汇总」的模式

**为什么重要**：
- 不能依赖短路行为来避免某些条件的副作用
- 异常会中断整个 map 过程，导致整个条件组求值失败（返回 false）
- 性能上：条件数较多时会有完整遍历开销

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

**完整求值流程**：
1. 调用 `evaluateConditionGroup` 处理最外层条件组
2. **先 map 遍历**所有子项，递归处理嵌套组或计算单个条件，**全部执行完**收集到 results 数组
3. **再用 every/some** 根据连接器汇总结果
4. ✅ 关键：任何异常都触发 catch，整个 `evaluateLogic` 返回 false

**单个条件求值（evaluateSingleCondition）**：
```
1. 获取左操作数值 → getLeftOperandValue()
2. 获取右操作数值（如需要） → getRightOperandValue()
3. 根据 operator 类型执行比较
4. 异常捕获：任何异常都返回 false
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
- 当左值是**长度为 1 的数组**，右值是**字符串**时，执行 `includes` 比较而非严格相等
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

**关键代码**：`packages/surveys/src/components/general/survey.tsx:698-806`

```typescript
// 逐条处理逻辑规则
for (const logic of currentBlock.logic) {
  const result = processLogicRule(logic, firstJumpTarget, allRequiredQuestionIds);
  firstJumpTarget = result.jumpTarget;
  // calculationResults 会被更新并传递给后续规则
  calculationResults = result.updatedCalculations;
}
```

**执行顺序详解**：

1. **串行执行**：同一 Block 下的多条逻辑规则按定义顺序**逐条串行执行**
2. **变量更新传播**：前一条规则的 calculate 结果会更新 `calculationResults`，影响后续规则的条件求值
3. **jump 优先级**：只取第一个满足条件的 jump target，后续规则的 jump 被静默忽略

### 3.3 容易误判的动作细节

#### ⚠️ 多个 jumpToBlock 时只取第一个
**代码位置**：`survey.tsx:751`
```typescript
const newJumpTarget = jumpTarget && !currentJumpTarget ? jumpTarget : currentJumpTarget;
```
- 如果条件组满足后触发多个 `jumpToBlock` 动作，**只有第一个会生效**
- 后续的 jump 动作会被**静默忽略**，无任何警告
- **设计意图**：避免跳转冲突，但缺乏冲突提示

#### ⚠️ 变量更新影响后续规则判断
**代码位置**：`survey.tsx:773`
```typescript
calculationResults = result.updatedCalculations;
```
- 前一条规则的 calculate 动作修改变量后，新值会**立即影响**后续规则的条件判断
- 这意味着：**规则定义顺序非常重要**，不同顺序可能导致完全不同的跳转结果

#### ➗ 除以零保护
**代码位置**：`packages/types/surveys/blocks.ts:48-54` 和 `logic.ts:505-506`
- 定义时验证：Zod schema 会拒绝静态值为 0 的除法
- 运行时保护：如果运行时计算出现除以零，**保持原始变量值不变**

#### 📊 计算操作不回滚
- calculate 动作按顺序执行
- 如果后续动作失败，之前的变量修改**不会回滚**
- 但由于异常被捕获并返回 false，整个条件组被视为不满足时，calculate 的副作用是否保留需要查看调用方实现

---

## 4. 最小示例：多条规则串行时变量更新改变后续规则命中

### 4.1 场景设定

假设 Block 中有 3 条规则，变量 `score` 初始值为 90：

| 规则 | 条件 | 动作 |
|-----|------|------|
| **规则 1** | `score < 100` | calculate: score += 15<br>jumpToBlock: "BlockA" |
| **规则 2** | `score >= 100` | jumpToBlock: "BlockB" |
| **规则 3** | `score >= 95` | jumpToBlock: "BlockC" |

### 4.2 真实执行过程

```
初始状态：score = 90, jumpTarget = undefined
──────────────────────────────────────────────

执行规则 1：
  条件判断：score < 100 → 90 < 100 → true ✅
  执行动作：
    calculate: score = 90 + 15 = 105
    jumpToBlock: jumpTarget = "BlockA"
  状态更新：score = 105, jumpTarget = "BlockA"

──────────────────────────────────────────────
执行规则 2（使用更新后的 score）：
  条件判断：score >= 100 → 105 >= 100 → true ✅
  执行动作：
    jumpToBlock: jumpTarget 已设置，忽略 → 保持 "BlockA"
  状态更新：score = 105, jumpTarget = "BlockA"（不变）

──────────────────────────────────────────────
执行规则 3（使用更新后的 score）：
  条件判断：score >= 95 → 105 >= 95 → true ✅
  执行动作：
    jumpToBlock: jumpTarget 已设置，忽略 → 保持 "BlockA"
  状态更新：score = 105, jumpTarget = "BlockA"（不变）

──────────────────────────────────────────────
最终结果：跳转目标 = "BlockA"
```

### 4.3 关键观察

1. **变量更新是即时的**：规则 1 的 calculate 结果立即影响规则 2 和规则 3 的条件判断
2. **规则 2 和 3 本也能命中**：score 更新到 105 后，两条规则的条件都满足了
3. **但 jump 只取第一个**：后续两条规则的 jumpToBlock 被静默忽略
4. **顺序敏感性**：如果把规则 2 放到规则 1 前面，结果会完全不同：
   - 规则 2 先执行：`90 >= 100` → false，不触发
   - 规则 1 再执行：`90 < 100` → true，最终还是跳 BlockA
   - 这个例子顺序调换结果不变，但其他场景可能不同

### 4.4 更有说服力的示例：反向顺序导致不同结果

| 规则 | 条件 | 动作 |
|-----|------|------|
| **规则 A** | `score < 100` | calculate: score = 200<br>jumpToBlock: "BlockA" |
| **规则 B** | `score == 90` | jumpToBlock: "BlockB" |

**顺序 A → B 的结果**：
- 规则 A：score=90 < 100 → true，score 变成 200，跳 BlockA
- 规则 B：score=200 == 90 → false，不触发
- **最终：BlockA**

**顺序 B → A 的结果**：
- 规则 B：score=90 == 90 → true，跳 BlockB（jump 已设置）
- 规则 A：score=90 < 100 → true，score 变成 200，但 jump 忽略
- **最终：BlockB**

✅ **结论**：规则定义顺序直接影响最终跳转结果！

---

## 5. 默认分支与边界情况（logicFallback）

### 5.1 默认分支机制
每个 Block 可以配置 `logicFallback` 字段，定义**所有逻辑规则都不满足时**的跳转目标。

```typescript
logicFallback?: string;  // 条件都不满足时跳转的 Block ID
```

### 5.2 完整的跳转决策流程

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
    │ 串行执行  │                   │
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
      │ jumpTarget  按顺序到下一个块 │
      │   │          │               │
      └───┴──────────┴───────────────┘
                    │
                    ▼
            检查 jumpTarget 有效性
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
    是合法 Block?        是 ending card?
          │                   │
      是 ─┴─ 否            是 ─┴─ 否
      │       │             │       │
      ▼       ▼             ▼       ▼
    跳转目标  ───────────► 问卷结束（结束态）
```

### 5.3 边界场景汇总

| 场景 | 行为 | 风险等级 |
|-----|------|---------|
| Block.logic 为空 | 按问卷定义顺序跳转到下一个 Block | 🟢 低 |
| 所有条件都不满足，无 logicFallback | 按顺序到下一个 Block | 🟢 低 |
| 所有条件都不满足，有 logicFallback | 跳转到 logicFallback 指定的块 | 🟡 中 |
| 条件求值发生异常 | 该条件返回 false，继续评估其他条件 | 🟠 高 |
| 满足多个逻辑规则 | 逐条执行所有满足的规则的动作 | 🟡 中 |
| 多个规则都触发 jumpToBlock | **所有规则中的第一个 jump 生效** | 🔴 严重 |
| jumpTarget 指向不存在的 Block | **问卷直接结束，进入结束态** | 🔴 严重 |
| jumpTarget 指向 ending card | 问卷直接结束 | 🟡 中 |
| 变量计算导致后续规则条件变化 | 后续规则使用更新后的变量值求值 | 🟠 高 |

---

## 6. 无效 jump 目标落到结束态的判定条件

### 6.1 判定代码
**代码位置**：`packages/surveys/src/components/general/survey.tsx:943-944`

```typescript
const { nextBlockId, calculatedVariables } = evaluateLogicAndGetNextBlockId(surveyResponseData);

const finished =
  // 条件 1：无跳转目标（undefined）
  nextBlockId === undefined ||
  // 条件 2：跳转目标不在已知 blocks 列表中
  !localSurvey.blocks.map((block) => block.id).includes(nextBlockId);

setIsSurveyFinished(finished);
```

### 6.2 触发结束态的两种情况

**情况 1：nextBlockId 为 undefined**
- 没有任何规则满足，且没有 logicFallback
- 已经是最后一个 Block，没有下一个
- 此时 `finished = true`，问卷结束

**情况 2：nextBlockId 不在 blocks 列表中**
- jumpTarget 引用了已被删除的 Block ID
- jumpTarget 格式错误或拼写错误
- jumpTarget 是 ending card ID（不在 blocks 中，属于独立的 endings 数组）
- 此时 `finished = true`，问卷直接结束

### 6.3 结束后的页面跳转逻辑
**代码位置**：`survey.tsx:964-974`

```typescript
if (nextBlockId) {
  // 即使 finished=true，如果有 nextBlockId 还是会尝试设置
  setBlockId(nextBlockId);
} else if (finished) {
  // 没有 nextBlockId 且判定为结束
  const firstEndingId = localSurvey.endings[0]?.id as string | undefined;
  if (firstEndingId) {
    // 有定义 ending card，跳转到第一个
    setBlockId(firstEndingId);
  } else {
    // 没有定义 ending，设置为 "end" 触发结束屏幕
    setBlockId("end");
  }
}
```

**⚠️ 注意**：即使 `finished=true`，只要有 `nextBlockId` 还是会尝试跳转到该 ID，哪怕是无效的。这可能导致问卷卡死或显示空白页面。

---

## 7. 关键风险点总结

### 🚨 高风险问题

#### 1. 异常静默吞噬
- 所有条件求值异常都返回 false，不暴露错误信息
- 可能导致：配置错误的逻辑长期不被发现
- **示例**：引用了已删除的问题 ID，该条件永久为 false，但无任何提示

#### 2. 条件组无短路执行
- 所有条件都会被求值，哪怕第一个已经能决定结果
- 可能导致：不必要的性能开销，异常影响范围扩大

#### 3. 多个 jump 冲突无提示
- 多个规则触发 jumpToBlock 时，只取第一个
- 无冲突警告，行为依赖规则定义顺序
- **示例**：
  ```
  规则1：A == 1 → 跳转到 BlockX
  规则2：B == 2 → 跳转到 BlockY
  如果 A==1 且 B==2，最终跳转到 BlockX（规则1优先）
  ```

#### 4. 数组相等比较反直觉
- `["a"] == "a"` 被判定为 true
- 可能导致多选问题的逻辑判断不符合预期

#### 5. jumpTarget 有效性无运行时验证
- 跳转到不存在的 Block ID 时无保护机制
- 直接导致问卷提前结束
- **判定条件**：`!localSurvey.blocks.map((block) => block.id).includes(nextBlockId)`

### ⚠️ 中等风险问题

#### 6. 变量计算的副作用
- 条件满足后执行 calculate 会修改变量值
- 变量值变化可能影响后续条件的求值
- **规则定义顺序直接影响最终结果**

#### 7. 多语言下的选项比较
- MultipleChoice 选项比较使用本地化标签（label），而非 ID
- 语言切换可能导致逻辑行为变化

#### 8. Matrix 行索引越界
- 行索引转换失败或超出范围时，静默返回 undefined
- 最终条件返回 false，难以调试

---

## 8. 操作符完整列表与注意事项

### 8.1 无需右操作数的操作符（12个）
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

### 8.2 需要右操作数的操作符（26个）

**比较类**：`equals`, `doesNotEqual`, `isGreaterThan`, `isLessThan`, `isGreaterThanOrEqual`, `isLessThanOrEqual`

**字符串操作**：`contains`, `doesNotContain`, `startsWith`, `doesNotStartWith`, `endsWith`, `doesNotEndWith`

**数组包含**：`equalsOneOf`, `includesAllOf`, `includesOneOf`, `doesNotIncludeAllOf`, `doesNotIncludeOneOf`, `isAnyOf`

**日期比较**：`isAfter`, `isBefore`

---

## 9. 调试与验证建议

### 9.1 开发调试建议
1. 在 `evaluateLogic` 和 `performActions` 中添加临时调试日志
2. 特别关注异常捕获点，建议增加开发环境的错误提示
3. 添加 jump 冲突检测警告
4. 增加 jumpTarget 有效性预检查

### 9.2 逻辑配置最佳实践

#### 🔑 规则顺序非常重要
- 把可能产生冲突的 jump 规则按优先级排列
- 变量计算规则放在前面，影响后续规则
- **避免**：同一 block 中多条规则都设置 jumpToBlock
- **谨记**：条件组求值无短路，所有条件都会执行

#### 🎯 避免循环依赖
- 不要让规则 A 修改变量 B，规则 B 又修改变量 A
- 可能导致难以预测的行为

#### 🛡️ 始终设置安全网
- 建议每个 Block 都配置 `logicFallback`
- 即使所有条件都不满足，也能控制跳转方向

#### 📋 避免引用不存在的目标
- 配置时验证 jump target 是否存在
- 删除 Block 时检查引用它的 logic
- 考虑在编辑器层面增加有效性检查

#### 🔢 Matrix 索引从 0 开始
- logic 中 row 索引是数字，从 0 开始
- 与 UI 显示的 1-based 序号不对应

---

## 10. 关键代码位置参考

| 功能模块 | 文件位置 | 关键行号 |
|---------|---------|---------|
| 条件组求值（map + every/some） | `packages/surveys/src/lib/logic.ts` | 35-45 |
| 动作执行 | `packages/surveys/src/lib/logic.ts` | 50-82 |
| 左操作数获取 | `packages/surveys/src/lib/logic.ts` | 84-181 |
| 单个条件求值 | `packages/surveys/src/lib/logic.ts` | 206-448 |
| Block 类型定义 | `packages/types/surveys/blocks.ts` | 124-151 |
| 逻辑类型定义 | `packages/types/surveys/logic.ts` | 1-246 |
| 跳转逻辑处理（规则串行） | `packages/surveys/src/components/general/survey.tsx` | 698-806 |
| 提交处理函数 | `packages/surveys/src/components/general/survey.tsx` | 919-995 |
| 结束态判定条件 | `packages/surveys/src/components/general/survey.tsx` | 943-944 |
