# Survey Logic 异常链路分析报告

## 1. 异常处理机制概览

### 1.1 关键发现

| 函数 | 是否有异常处理 | 处理方式 |
|------|--------------|----------|
| **evaluateLogic** | ❌ **无** | 直接调用，无 try-catch 包裹 |
| **evaluateSingleCondition** | ✅ 有 | 内部 try-catch，异常时返回 `false` |

---

## 2. 异常处理代码细节

### 2.1 evaluateLogic - 无顶层异常兜底

```typescript
// packages/surveys/src/lib/logic.ts:28-48
export const evaluateLogic = (
  localSurvey: TJsEnvironmentStateSurvey,
  data: TResponseData,
  variablesData: TResponseVariables,
  conditions: TConditionGroup,
  selectedLanguage: string
): boolean => {
  const evaluateConditionGroup = (group: TConditionGroup): boolean => {
    const results = group.conditions.map((condition) => {
      if (isConditionGroup(condition)) {
        return evaluateConditionGroup(condition);  // 递归调用，无异常处理
      } else {
        return evaluateSingleCondition(
          localSurvey, data, variablesData, condition, selectedLanguage
        );
      }
    });

    return group.connector === "or" 
      ? results.some((r) => r)  // OR 逻辑：任一为 true
      : results.every((r) => r); // AND 逻辑：全部为 true
  };

  return evaluateConditionGroup(conditions);
  // ❌ 无 try-catch！整个函数没有异常兜底
};
```

### 2.2 evaluateSingleCondition - 唯一的异常兜底点

```typescript
// packages/surveys/src/lib/logic.ts:206-448
const evaluateSingleCondition = (
  localSurvey: TJsEnvironmentStateSurvey,
  data: TResponseData,
  variablesData: TResponseVariables,
  condition: TSingleCondition,
  selectedLanguage: string
): boolean => {
  try {
    // --- 条件值读取 ---
    let leftValue = getLeftOperandValue(...);  // 可能抛出异常
    let rightValue = condition.rightOperand 
      ? getRightOperandValue(...) 
      : undefined;

    // --- 类型安全检查 ---
    let leftField = ...;  // 可能访问 undefined 属性
    let rightField = ...; // 可能访问 undefined 属性

    // --- 类型转换与特殊处理 ---
    if (condition.leftOperand.type === "variable" && ...) {
      rightValue = Number(rightValue as string); // 可能 NaN
    }

    // --- 40+ 种操作符逻辑 ---
    switch (condition.operator) {
      case "equals": /* ... */
      case "contains": /* ... */
      case "isSubmitted": /* ... */
      case "isGreaterThan": /* ... */
      // ... 40+ 种操作符
      default:
        return false;
    }
  } catch (e) {
    // ✅ 唯一的异常捕获点：吞掉所有异常，静默返回 false
    return false;
  }
};
```

---

## 3. 真实调用链路：从条件读取失败到最终分支选择

### 3.1 完整调用链

```
用户提交答案 (onSubmit)
    ↓
evaluateLogicAndGetNextBlockId()  [survey.tsx:698-806]
    ├─> 构建 localResponseData（合并当前答案）
    ├─> 构建 calculationResults（变量计算结果）
    └─> processLogicRule()
         └─> evaluateLogic()  [logic.ts:28]
              └─> evaluateConditionGroup()
                   └─> group.conditions.map()
                        ├─> [嵌套条件组] → 递归 evaluateConditionGroup
                        └─> [单个条件] → evaluateSingleCondition()
                             ├─> getLeftOperandValue()  ← 异常可能发生
                             ├─> getRightOperandValue() ← 异常可能发生
                             ├─> [类型转换]
                             ├─> [操作符逻辑]
                             └─> 捕获异常 → return false  ← 异常被吞掉
         ├─> 检查 isLogicMet (true/false)
         ├─> true: 执行 performActions() → 计算 jumpTarget
         └─> false: 不触发动作，使用 logicFallback
    ├─> 无 jumpTarget 时使用 logicFallback
    └─> 决定 nextBlockId
         ├─> jumpTarget（来自逻辑动作）
         └─> localSurvey.blocks[currentBlockIndex + 1]?.id
```

### 3.2 关键分支选择逻辑

```typescript
// survey.tsx:723-760
const processLogicRule = (...) => {
  const isLogicMet = evaluateLogic(...);  // 异常时返回 false

  if (!isLogicMet) {
    // ❌ 条件不满足（或异常），不执行任何动作
    return { jumpTarget: currentJumpTarget, ... };
  }

  // ✅ 条件满足，执行动作（跳转、设置必填等）
  const { jumpTarget, requiredQuestionIds, calculations } = 
    performActions(localSurvey, logic.actions, localResponseData, ...);
  // ...
};
```

**异常时的分支选择结果**：
- 当 `evaluateSingleCondition` 内部发生异常 → 返回 `false`
- 条件组结果：`AND` 逻辑下整个条件组为 `false`；`OR` 逻辑下取决于其他条件
- 若整个逻辑规则结果为 `false` → **不执行任何动作**，使用 `logicFallback`（通常是顺序下一题）

---

## 4. 误判反例分析

### 4.1 反例场景 1：变量不存在导致异常

**问题条件配置**：
```json
{
  "connector": "and",
  "conditions": [
    {
      "leftOperand": { "type": "variable", "value": "non_existent_var" },
      "operator": "equals",
      "rightOperand": { "type": "static", "value": "100" }
    }
  ]
}
```

**异常链路**：
1. `evaluateSingleCondition` 调用 `getLeftOperandValue`
2. 变量 `non_existent_var` 不存在 → `getVariableValue` 返回 `undefined`
3. 执行 `Number(undefined)` → `NaN`
4. 后续比较逻辑可能抛出异常（取决于操作符）
5. **被 catch 捕获 → 返回 `false`**
6. 整个条件组结果为 `false`
7. **不执行跳转动作，使用 logicFallback 顺序前进**

**预期行为 vs 实际行为**：
- ✅ 预期：变量不存在时应该如何处理？（配置错误处理）
- ❌ 实际：静默返回 false，条件不触发，用户按正常流程继续

---

### 4.2 反例场景 2：矩阵题引用不存在的行

**问题条件配置**：
```json
{
  "connector": "and",
  "conditions": [
    {
      "leftOperand": { 
        "type": "element", 
        "value": "matrix_question_id",
        "meta": { "row": "999" }  // 第 999 行，实际只有 5 行
      },
      "operator": "equals",
      "rightOperand": { "type": "static", "value": "Option A" }
    }
  ]
}
```

**异常链路**：
```typescript
// getLeftOperandValue 内部（logic.ts:148-168）
if (leftOperand.meta && leftOperand.meta?.row !== undefined) {
  const rowIndex = Number(leftOperand.meta.row);  // rowIndex = 999
  
  // ❌ rowIndex 999 >= currentQuestion.rows.length
  if (isNaN(rowIndex) || rowIndex < 0 || rowIndex >= currentQuestion.rows.length) {
    return undefined;  // ✅ 正常路径返回 undefined，不会异常
  }
  
  const row = getLocalizedValue(currentQuestion.rows[rowIndex].label, selectedLanguage);
  // 当 rowIndex 超界时，如果前面检查被绕过（类型转换问题）→ 访问 undefined.label 抛异常
  // ❌ 被 catch 捕获 → 返回 false
}
```

**误判结果**：
- 真实情况：配置错误，引用了不存在的行
- 系统行为：静默返回 `false`，条件不触发
- **风险**：管理员无法发现配置错误，业务逻辑静默失效

---

### 4.3 反例场景 3：日期格式解析异常

**问题条件配置**：
```json
{
  "connector": "and",
  "conditions": [
    {
      "leftOperand": { "type": "element", "value": "date_question" },
      "operator": "equals",
      "rightOperand": { "type": "static", "value": "not-a-date" }  // 非法日期格式
    }
  ]
}
```

**异常链路**：
```typescript
case "equals":
  if (condition.leftOperand.type === "element") {
    if ((leftField as TSurveyElement).type === TSurveyElementTypeEnum.Date) {
      // ❌ new Date("not-a-date").getTime() 返回 NaN
      return new Date(leftValue).getTime() === new Date(rightValue).getTime();
      // NaN === NaN → false（但这是正常逻辑，不是异常）
    }
  }
```

**特殊说明**：日期格式问题通常不会触发异常，而是正常返回 `false`，因为 `NaN === NaN` 本身就是 `false`。

---

### 4.4 反例场景 4：嵌套条件组中的异常传播

**问题条件配置**：
```json
{
  "connector": "or",  // OR 逻辑
  "conditions": [
    { "connector": "and", "conditions": [/* 正常条件 A */] },  // → true
    { /* 条件 B，会抛出异常 */ },  // → 被捕获返回 false
    { "connector": "and", "conditions": [/* 正常条件 C */] }   // → true
  ]
}
```

**结果计算**：
- 条件 A: true
- 条件 B: false（异常被捕获）
- 条件 C: true
- OR 逻辑结果: `true || false || true` = **true**

**误判结果**：
- 条件 B 发生了异常，但由于 OR 逻辑中其他条件为 true，整个规则仍然触发
- **静默异常无法被发现**，业务逻辑看似正常工作

---

## 5. 风险总结与影响面

| 风险场景 | 影响程度 | 可观测性 | 说明 |
|---------|---------|---------|------|
| 条件配置错误（变量不存在、行列超界） | 中 | ❌ 不可观测 | 静默返回 false，管理员无法发现问题 |
| 嵌套条件组中部分条件异常 | 低/中 | ❌ 不可观测 | 取决于其他条件的结果，可能掩盖问题 |
| 数据类型转换异常（Number/Date） | 中 | ❌ 不可观测 | 正常路径 vs 异常路径结果可能相同 |
| 操作符逻辑边缘情况异常 | 低 | ❌ 不可观测 | 极端数据触发的异常被吞掉 |

### 5.1 关键架构决策权衡

**决策**：`evaluateSingleCondition` 内 try-catch 所有异常，返回 false

**理由**：
- ✅ 保证问卷流程不中断，用户体验连续
- ✅ 避免单个条件错误导致整个问卷无法提交
- ❌ 丧失可观测性，配置错误难以发现
- ❌ 异常原因被掩盖，调试困难

**建议改进**：
1. 开发环境下在控制台输出警告日志
2. 增加配置验证阶段，提前检测无效条件引用
3. 考虑区分"逻辑结果为 false"和"异常导致的 false"
