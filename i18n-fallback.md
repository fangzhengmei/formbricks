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

## 二、链接问卷语言协商流程

### 2.1 URL lang 参数处理

**核心逻辑**（`apps/web/modules/survey/link/components/survey-renderer.tsx:203-217`）：

```typescript
function getLanguageCode(langParam: string | undefined, survey: TSurvey): string {
  if (!langParam) return "default";

  const selectedLanguage = survey.languages.find((surveyLanguage) => {
    return (
      surveyLanguage.language.code.toLowerCase() === langParam.toLowerCase() ||
      surveyLanguage.language.alias?.toLowerCase() === langParam.toLowerCase()
    );
  });

  if (!selectedLanguage || selectedLanguage?.default || !selectedLanguage?.enabled) {
    return "default";
  }
  return selectedLanguage.language.code;
}
```

### 2.2 别名匹配机制

- **别名定义**：每个语言可配置自定义别名（如 `en` → `English`，`fr` → `Français`）
- **匹配规则**：URL `lang` 参数同时匹配 `code` 和 `alias`，大小写不敏感
- **别名验证**：管理端确保别名不与 ISO 639 标准代码冲突，避免歧义
- **典型用途**：
  - 用友好的语言名称作为 URL 参数（如 `?lang=English`）
  - 兼容旧版链接的语言参数
  - 多区域语言变体映射（如 `fr-FR` → `fr`）

### 2.3 禁用语言与默认语言回退判定

回退触发条件（满足任一即回退到 `default`）：

| 条件 | 说明 |
|-----|------|
| `!langParam` | URL 未提供 lang 参数 |
| `!selectedLanguage` | 找不到匹配的语言（code 或 alias 都不匹配） |
| `selectedLanguage.default` | 匹配到的是默认语言，直接用 default 标识 |
| `!selectedLanguage.enabled` | 匹配到的语言已被管理员禁用 |

> **注意**：`default` 是 Formbricks 内部标识符，不是真实语言代码。它表示"使用问卷管理员设置的默认语言"。

### 2.4 浏览器语言自动匹配

当 URL 未提供 `lang` 参数时，通过 `Accept-Language` 头自动匹配：

**位置**：`apps/web/lib/utils/locale.ts:5-29`

匹配优先级：
1. 精确匹配完整 locale（如 `zh-CN`）
2. 前缀匹配语言代码（如 `zh` → `zh-Hans-CN`）
3. 回退到系统默认语言 `DEFAULT_LOCALE`

---

## 三、字段级回退机制

### 3.1 管理端 vs 答题端：两个不同的实现

Formbricks 存在两个独立的 `getLocalizedValue` 函数，回退行为**完全不同**：

| 维度 | 答题端（受访者可见） | 管理端（编辑/预览） |
|-----|---------------------|---------------------|
| 位置 | `packages/surveys/src/lib/i18n.ts` | `apps/web/lib/i18n/utils.ts` |
| 回退行为 | 目标语言缺失 → 回退 `default` | 目标语言缺失 → 返回空字符串 |
| 设计目标 | 确保受访者总能看到内容 | 让编辑者清晰看到哪些翻译缺失 |

**答题端实现**（受访者看到的问卷）：
```typescript
// packages/surveys/src/lib/i18n.ts:15-37
export const getLocalizedValue = (value: TI18nString | undefined, languageId: string): string => {
  if (!value) return "";
  if (isI18nObject(value)) {
    if (typeof value[languageId] === "string") {
      return value[languageId];  // 优先级 1: 目标语言
    } else {
      return value.default;       // 优先级 2: 问卷默认语言（关键差异）
    }
  }
  return "";
};
```

**管理端实现**（编辑/分析界面）：
```typescript
// apps/web/lib/i18n/utils.ts:57-68
export const getLocalizedValue = (value: TI18nString | undefined, languageId: string): string => {
  if (!value) return "";
  if (isI18nObject(value)) {
    if (value[languageId]) {
      return value[languageId];  // 优先级 1: 目标语言
    }
    return "";                   // 优先级 2: 空字符串（关键差异）
  }
  return "";
};
```

### 3.2 差异设计的合理性

| 场景 | 行为 | 原因 |
|-----|------|------|
| 受访者答题 | 显示 `default` 内容 | 避免空值造成糟糕的用户体验，确保问卷可用 |
| 管理员编辑 | 显示空值 | 直观提示哪些翻译尚未完成，驱动补全翻译 |
| 数据分析 | 显示空值 | 确保数据导出/展示时能准确反映翻译状态 |

### 3.3 空字符串 vs undefined 处理

两个实现的共同行为：
- `value === undefined` → 返回 `""`
- `value[languageId]` 是 `undefined`/`null`/`false` → 触发各自的回退逻辑
- `value[languageId]` 是空字符串 `""` → **直接返回空字符串，不触发回退**

> **重要陷阱**：如果某个语言的翻译被显式设置为空字符串，答题端也会显示空值，而不会回退到 `default`。这是"翻译已被刻意清空"和"翻译尚未添加"的语义区别。

---

## 四、变量插值的二级回退

在 Recall（变量回填）场景中，存在独立于语言回退的机制：

**位置**：`packages/surveys/src/lib/recall.ts`

**语法**：`#recall:questionId/fallback:默认值#`

```typescript
modifiedText = modifiedText.replace(recallInfo, value?.toString() || fallback);
```

回退优先级（从高到低）：
1. 真实用户回答值（从 `responseData` 或 `variables` 获取）
2. Recall 语法中 `/fallback:` 后定义的默认值
3. 空字符串（如果连 fallback 都未提供）

**示例**：
```
感谢您对 "#recall:q1/fallback:我们的产品#" 的评价！
```
- 如果 q1 有答案 → 显示答案
- 如果 q1 无答案 → 显示"我们的产品"
- 如果无 fallback → 显示空字符串

---

## 五、完整回退链总结

### 5.1 链接问卷语言决策流程

```
受访者访问链接 /s/abc123?lang=XX
              ↓
┌─────────────────────────────────────────────┐
│ 1. URL lang 参数解析                        │
│    - 精确匹配 language.code                │
│    - 别名匹配 language.alias               │
│    - 大小写不敏感                           │
└─────────────────────────────────────────────┘
              ↓ 匹配失败 / 语言禁用 / 是默认语言
┌─────────────────────────────────────────────┐
│ 2. 回退到 "default" 标识                    │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│ 3. "default" 解析为问卷默认语言的真实 code  │
│    (由 I18nProvider 在渲染时完成)           │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│ 4. 字段级翻译查找                           │
│    - 答题端: 目标语言 → default → ""       │
│    - 管理端: 目标语言 → ""                 │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│ 5. Recall 变量插值                          │
│    - 真实值 → fallback → ""                │
└─────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────┐
│ 6. UI 界面元素翻译                          │
│    - i18next: 目标语言 → en                │
└─────────────────────────────────────────────┘
```

### 5.2 统一优先级总表

| 层级 | 回退触发条件 | 回退链（从高到低） | 实现位置 |
|-----|-------------|-------------------|---------|
| **语言协商** | URL 参数无效 / 语言禁用 / 是默认语言 | lang 参数匹配 → `"default"` 标识 | `survey-renderer.tsx` |
| **答题端字段** | 目标语言翻译不存在/undefined | 目标语言 → `value.default` → `""` | `packages/surveys/src/lib/i18n.ts` |
| **管理端字段** | 目标语言翻译不存在/undefined | 目标语言 → `""`（无 default 回退） | `apps/web/lib/i18n/utils.ts` |
| **Recall 变量** | 问题无答案 / 变量不存在 | 真实值 → `fallback:xxx` → `""` | `packages/surveys/src/lib/recall.ts` |
| **UI 元素翻译** | i18next 无对应翻译 | 目标语言 → `"en"` | `i18n.config.ts` |

---

## 六、设计要点与注意事项

### 6.1 核心设计原则

1. **双轨制回退**：问卷内容使用 `default` 键回退（业务可控），UI 使用硬编码 `en` 回退（技术稳定）

2. **双版本实现**：答题端保证可用性（回退 default），管理端保证可见性（不回退，显示空值提示补全）

3. **"default" 不是真实语言**：它是一个指针，指向问卷管理员设置的默认语言，实际渲染时会解析为真实语言代码

4. **字段级独立回退**：每个字段独立判断是否需要回退，避免因单个字段缺译导致整个语言版本不可用

### 6.2 常见陷阱

| 陷阱 | 现象 | 规避方法 |
|-----|------|---------|
| 显式空字符串不回退 | 某语言翻译被设为 `""`，受访者看到空白 | 在管理端验证时，检查启用语言的字段不能为空白 |
| 禁用语言不报错 | `?lang=disabled` 静默回退默认语言 | 分享链接时只提供启用语言的选项 |
| alias 与 code 冲突 | 别名与其他语言 code 相同导致匹配歧义 | 管理端已做验证，禁止设置冲突的别名 |
| `default` 语言代码 | 不要在 `TI18nString` 中添加 `{ default: "...", default: "..." }`，`default` 是键名不是语言代码 | 问卷默认语言的真实内容存储在 `default` 键下 |

### 6.3 典型场景示例

**场景**：问卷默认语言为英语（`en`），已添加法语（`fr`）但标题翻译为空，德语（`de`）已被管理员禁用。

| 访问链接 | 受访者看到的标题 | 说明 |
|---------|-----------------|------|
| `/s/abc?lang=en` | 英语标题 | 精确匹配，正常显示 |
| `/s/abc?lang=fr` | **空字符串** | 法语翻译被显式设为空，不回退 |
| `/s/abc?lang=de` | 英语标题（default） | 德语被禁用，回退到默认语言 |
| `/s/abc?lang=invalid` | 英语标题（default） | 语言不存在，回退默认 |
| `/s/abc` | 英语标题（default） | 无 lang 参数，回退默认 |
