# Formbricks i18n 翻译回退机制报告

## 一、翻译资源结构

Formbricks 的翻译系统分为两层，采用不同的资源组织方式：

### 1.1 问卷 UI 界面翻译（系统级）

**位置**：`packages/surveys/locales/` 目录

**特点**：
- 采用标准的 i18next 资源格式，预编译打包
- 支持 19 种语言：ar, da, de, en, es, et, fr, hi, hu, it, ja, nl, pt, ro, ru, sv, tr, uz, zh-Hans
- 所有翻译资源在 `i18n.config.ts` 中静态导入，初始化时全部加载

**配置示例**（`packages/surveys/src/lib/i18n.config.ts`）：
```typescript
i18n.init({
  fallbackLng: "en",  // 系统级回退语言
  supportedLngs: [...],
  resources: {
    en: { translation: enTranslations },
    zh: { translation: zhHansTranslations },
    // ...
  }
})
```

### 1.2 问卷内容翻译（业务级）

**类型定义**：`packages/types/i18n.ts`
```typescript
export type TI18nString = Record<string, string> & { default: string };
```

**特点**：
- 每个可翻译字段都是一个独立的多语言对象
- **必须包含 `default` 键**，作为字段级别的默认回退
- 其他键为语言代码（如 `en`, `zh-Hans`, `fr` 等）
- 存储在数据库中，随问卷数据一起加载

**示例**：
```typescript
const headline: TI18nString = {
  default: "How likely are you to recommend us?",  // 问卷默认语言
  en: "How likely are you to recommend us?",
  zh: "您向我们推荐的可能性有多大？",
  fr: "Quelle est la probabilité que vous nous recommandiez ?"
};
```

---

## 二、语言协商流程

### 2.1 语言标识符体系

系统中存在三种语言标识：

| 标识类型 | 示例 | 说明 |
|---------|------|------|
| 问卷默认语言 | `"default"` | Formbricks 内部标识符，**不是有效的 i18next locale**，表示使用问卷管理员设置的默认语言 |
| 标准语言代码 | `"en"`, `"zh-Hans"` | i18next 支持的语言代码，用于 UI 翻译 |
| Web App locale | `"en-US"`, `"zh-Hans-CN"` | Web 管理后台使用的完整 locale 格式 |

### 2.2 语言解析优先级

```
受访者浏览器语言
    ↓
┌─────────────────────────────────┐
│ 检查问卷是否支持该语言          │
└─────────────────────────────────┘
    ↓ 不支持
┌─────────────────────────────────┐
│ 使用问卷默认语言 ("default")    │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ 映射到 i18next 语言代码         │
│ （"default" → 问卷默认语言代码）│
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│ i18next 系统级回退 → "en"      │
└─────────────────────────────────┘
```

### 2.3 关键实现点

**1) "default" 特殊处理**（`packages/surveys/src/lib/i18n.ts:49`）：
```typescript
const resolvedCode = languageCode === "default" ? i18n.language : languageCode;
```
> 注意：当传入 `"default"` 时，使用 i18n 实例当前已解析的语言，而不是强制改回 "en"，避免覆盖用户的语言选择（Issue #7515）。

**2) 语言代码映射**（`apps/web/modules/survey/link/lib/utils.ts:62`）：
```typescript
const languageToLocaleMap: Record<string, string> = {
  en: "en-US",
  zh: "zh-Hans-CN",
  "zh-Hans": "zh-Hans-CN",
  // ...
};
```

---

## 三、字段级回退机制

### 3.1 核心函数：`getLocalizedValue`

**位置**：`packages/surveys/src/lib/i18n.ts:15`

```typescript
export const getLocalizedValue = (
  value: TI18nString | undefined,
  languageId: string,
  replaceNewLines: boolean = false
): string => {
  if (!value) return "";
  
  if (isI18nObject(value)) {
    // 优先级 1: 目标语言存在 → 使用目标语言
    if (typeof value[languageId] === "string") {
      result = value[languageId];
    } 
    // 优先级 2: 目标语言缺失 → 回退到 default（问卷默认语言）
    else {
      result = value.default;
    }
  }
  return result;
};
```

### 3.2 字段级回退优先级

```
目标语言（如 "fr"）
    ↓ 不存在
问卷默认语言（"default" 键）
    ↓ 字段为 undefined
空字符串 ""
```

> **重要区别**：
> - 系统级（UI）：目标语言 → `en`（硬编码）
> - 字段级（问卷内容）：目标语言 → `default`（问卷管理员设置的默认语言）

### 3.3 变量插值的二级回退

在 Recall（变量回填）场景中，存在独立的回退机制：

**位置**：`packages/surveys/src/lib/recall.ts`

**语法**：`#recall:questionId/fallback:默认值#`

```typescript
export const replaceRecallInfo = (text, responseData, variables, languageCode) => {
  // ...
  // 优先级 1: 从 responseData 或 variables 获取真实值
  // 优先级 2: 使用 recall 语法中的 fallback 值
  // 优先级 3: fallback 值为空 → 显示空字符串
  modifiedText = modifiedText.replace(recallInfo, value?.toString() || fallback);
};
```

**示例**：
```
感谢您对 "#recall:q1/fallback:我们的产品#" 的评价！
```
- 如果 q1 有答案 → 显示答案
- 如果 q1 无答案 → 显示 "我们的产品"
- 如果无 fallback → 显示空字符串

---

## 四、完整回退链总结

### 4.1 问卷内容显示流程

```
┌─────────────────────────────────────────────────────────┐
│                    受访者访问问卷                         │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│ 1. 语言协商                                             │
│    - 首选：受访者浏览器语言                              │
│    - 回退：问卷默认语言 ("default")                     │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│ 2. 字段级翻译查找                                       │
│    - 首选：目标语言字段（如 headline["fr"]）            │
│    - 回退：headline["default"]（问卷默认语言内容）      │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Recall 变量插值                                       │
│    - 首选：用户已回答的问题值 / 传入变量                 │
│    - 回退：#recall:.../fallback:xxx# 中的默认值         │
└─────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────┐
│ 4. UI 界面元素翻译                                       │
│    - 首选：i18next 当前语言                              │
│    - 回退：i18next fallbackLng → "en"                   │
└─────────────────────────────────────────────────────────┘
```

### 4.2 回退机制对比表

| 层级 | 回退触发条件 | 回退目标 | 实现位置 |
|------|-------------|---------|---------|
| 语言协商 | 问卷不支持受访者语言 | 问卷默认语言 | `I18nProvider` + 问卷配置 |
| 字段级翻译 | 该字段无目标语言翻译 | 字段的 `default` 键 | `getLocalizedValue` |
| Recall 变量 | 问题无答案 / 变量不存在 | `fallback:` 后的值 | `replaceRecallInfo` |
| UI 界面翻译 | i18next 无对应翻译 | `"en"` | `i18n.config.ts` |

---

## 五、设计要点

1. **双轨制回退**：问卷内容使用 `default` 键回退（业务可控），UI 使用硬编码 `en` 回退（技术稳定）

2. **"default" 不是真实语言**：它是一个指针，指向问卷管理员设置的默认语言，实际渲染时会解析为真实语言代码

3. **字段级独立回退**：每个字段独立判断是否需要回退，避免因单个字段缺译导致整个语言版本不可用

4. **Recall 回退独立**：变量插值的回退与语言回退解耦，允许编辑者为每个变量单独设置默认值
