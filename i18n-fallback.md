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

## 二、链接问卷语言协商主链

### 2.1 核心逻辑：getLanguageCode

**唯一入口**：URL `lang` 查询参数是问卷内容语言选择的**唯一输入源**

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

### 2.2 语言匹配流程

```
输入：URL lang 参数（如 ?lang=fr）
          ↓
┌─────────────────────────────────────────────┐
│ 决策点 1: 是否有 lang 参数？                │
│  - 无 → 直接返回 "default"                 │
│  - 有 → 继续匹配                           │
└─────────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────────┐
│ 决策点 2: code 或 alias 匹配？              │
│  - 精确匹配 language.code（大小写不敏感）  │
│  - 精确匹配 language.alias（大小写不敏感） │
│  - 都不匹配 → 返回 "default"               │
└─────────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────────┐
│ 决策点 3: 语言有效性检查                    │
│  - 是否是默认语言？ → 返回 "default"       │
│  - 是否已被禁用？ → 返回 "default"         │
│  - 都通过 → 返回真实 language.code         │
└─────────────────────────────────────────────┘
          ↓
输出：真实语言代码 或 "default" 标识
```

### 2.3 别名匹配机制

- **别名定义**：每个语言可配置自定义别名（如 `en` → `English`，`fr` → `Français`）
- **匹配规则**：URL `lang` 参数同时匹配 `code` 和 `alias`，大小写不敏感
- **别名验证**：管理端确保别名不与 ISO 639 标准代码冲突，避免歧义
- **典型用途**：
  - 用友好的语言名称作为 URL 参数（如 `?lang=English`）
  - 兼容旧版链接的语言参数
  - 多区域语言变体映射（如 `fr-FR` → `fr`）

### 2.4 回退到 "default" 的触发条件

满足以下**任一条件**即回退到 `default` 标识：

| 条件 | 说明 |
|-----|------|
| `!langParam` | URL 未提供 lang 参数 |
| `!selectedLanguage` | 找不到匹配的语言（code 或 alias 都不匹配） |
| `selectedLanguage.default` | 匹配到的是问卷的默认语言，直接用 default 标识 |
| `!selectedLanguage.enabled` | 匹配到的语言已被管理员禁用 |

> **注意**：`default` 是 Formbricks 内部标识符，不是真实语言代码。它表示"使用问卷管理员设置的默认语言"。

---

## 三、Accept-Language 的作用边界

### 3.1 关键澄清：不是 metadata 语言主路径

**重要修正**：浏览器 `Accept-Language` **不决定页面 metadata 的语言**。

**metadata 语言的真实路径**：
1. **输入源**：URL `lang` 参数（`generateMetadata` 中第 37 行直接读取）
2. **处理**：`getMetadataForLinkSurvey(surveyId, languageCode)`
3. **定位**：`getBasicSurveyMetadata` 中 `getLocalizedValue(metadata.title, langCode)`
4. **结果**：页面标题、描述等 metadata 的语言由 URL `lang` 参数决定

**Accept-Language 不参与 metadata 的语言决策**。

### 3.2 Accept-Language 在 Link 页面的真实落点

**获取位置**：`apps/web/lib/utils/locale.ts` 中的 `findMatchingLocale()`

**真实用途**（仅传递给以下场景）：

| 场景 | 传递路径 | 作用 |
|-----|---------|------|
| **邮件验证流程** | `locale` prop → `<VerifyEmail locale={locale} />` | 邮件验证页面的本地化（非问卷内容） |
| **页面级格式化** | 作为 `TUserLocale` 类型传递 | 日期、时间、数字的格式化规则（非问卷内容） |
| **系统消息文案** | 潜在的页面级提示文案 | 当问卷不可用/出错时的系统消息语言 |

**匹配优先级**（`findMatchingLocale` 内部）：
1. 精确匹配完整 locale（如 `zh-CN`）
2. 前缀匹配语言代码（如 `zh` → `zh-Hans-CN`）
3. 回退到系统默认语言 `DEFAULT_LOCALE`

### 3.3 架构设计意图

| 决策维度 | URL lang 参数 | Accept-Language |
|---------|--------------|----------------|
| **控制范围** | 问卷内容、metadata 标题/描述 | 邮件验证页面、系统错误消息、日期格式 |
| **可控性** | 分享链接时显式指定，可预测 | 依赖用户浏览器设置，不可控 |
| **一致性保证** | 同一链接所有用户看到相同语言的问卷内容 | 不同用户可能看到不同语言的系统消息 |
| **设计目的** | 保证问卷分享体验的一致性 | 优化周边系统流程的用户体验 |

> **架构原则**：问卷核心内容由 URL 显式控制（保证分享一致性），仅周边辅助流程使用浏览器语言偏好。

---

## 四、字段级回退机制

### 4.1 管理端 vs 答题端：两个不同的实现

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

### 4.2 差异设计的合理性

| 场景 | 行为 | 原因 |
|-----|------|------|
| 受访者答题 | 显示 `default` 内容 | 避免空值造成糟糕的用户体验，确保问卷可用 |
| 管理员编辑 | 显示空值 | 直观提示哪些翻译尚未完成，驱动补全翻译 |
| 数据分析 | 显示空值 | 确保数据导出/展示时能准确反映翻译状态 |

### 4.3 空字符串 vs undefined 处理

两个实现的共同行为：
- `value === undefined` → 返回 `""`
- `value[languageId]` 是 `undefined`/`null`/`false` → 触发各自的回退逻辑
- `value[languageId]` 是空字符串 `""` → **直接返回空字符串，不触发回退**

> **重要陷阱**：如果某个语言的翻译被显式设置为空字符串，答题端也会显示空值，而不会回退到 `default`。这是"翻译已被刻意清空"和"翻译尚未添加"的语义区别。

---

## 五、变量插值的二级回退

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

## 六、双决策流总表

### 6.1 问卷内容语言主决策流

| 层级 | 输入源 | 决策逻辑 | 回退终点 | 实现位置 |
|-----|--------|---------|---------|---------|
| **1. 问卷语言协商** | URL `lang` 参数 | 匹配 code/alias → 检查 enabled/default | 真实语言代码 或 `"default"` 标识 | `survey-renderer.tsx:getLanguageCode` |
| **2. 答题端字段翻译** | 上一层输出的 languageId | 目标语言存在？ → 是则返回 | `value.default` → `""` | `packages/surveys/src/lib/i18n.ts` |
| **3. 管理端字段翻译** | 编辑界面选中的 languageId | 目标语言存在？ → 是则返回 | `""`（无 default 回退） | `apps/web/lib/i18n/utils.ts` |
| **4. Recall 变量插值** | 已翻译的文本内容 | 找到 recall 语法 → 尝试替换 | 真实值 → `fallback:xxx` → `""` | `packages/surveys/src/lib/recall.ts` |
| **5. UI 界面元素翻译** | i18next 当前语言 | 翻译资源存在？ → 是则返回 | `"en"` | `i18n.config.ts` |

### 6.2 页面 locale 辅助决策流

| 层级 | 输入源 | 决策逻辑 | 回退终点 | 实现位置 |
|-----|--------|---------|---------|---------|
| **A1. 浏览器语言识别** | `Accept-Language` 请求头 | 解析浏览器语言偏好列表 | 解析后的语言列表 | `apps/web/lib/utils/locale.ts` |
| **A2. Locale 精确匹配** | 上一步的语言列表 | 与系统支持的 locales 精确匹配 | 匹配成功的完整 locale | `apps/web/lib/utils/locale.ts` |
| **A3. 前缀匹配回退** | 未精确匹配的语言代码 | 取语言前缀匹配（如 `zh` → `zh-Hans-CN`） | 前缀匹配成功的 locale | `apps/web/lib/utils/locale.ts` |
| **A4. 系统默认回退** | 全部匹配失败 | 使用硬编码的 `DEFAULT_LOCALE` | `en-US` | `apps/web/lib/constants` |
| **A5. 邮件验证页面** | 上一步输出的 `locale` | 传递给 `<VerifyEmail>` 组件 | 邮件验证流程的本地化 | `survey-renderer.tsx` |

> **核心区分**：**主决策流**决定问卷内容（标题、问题、选项）的显示语言，由 URL 完全控制；**辅助决策流**决定周边系统页面（邮件验证、错误提示）的本地化，由浏览器语言隐式决定。

### 6.3 Link 页面完整数据流

```
受访者访问 /s/abc123?lang=fr
     │
     ├─────────────────────────────────────────────┐
     │ 问卷内容主决策流（由 ?lang=fr 决定）        │
     │  ┌──────────────────────────────────────┐  │
     │  │ getLanguageCode(lang="fr")           │  │
     │  │ → 匹配成功，输出 languageCode="fr"   │  │
     │  └──────────────────┬───────────────────┘  │
     │                     ↓                       │
     │  ┌──────────────────────────────────────┐  │
     │  │ 答题端 getLocalizedValue("fr")       │  │
     │  │ → 显示法语问卷内容                   │  │
     │  └──────────────────┬───────────────────┘  │
     │                     ↓                       │
     │  ┌──────────────────────────────────────┐  │
     │  │ getMetadataForLinkSurvey("fr")       │  │
     │  │ → 页面标题/描述使用法语              │  │
     │  └──────────────────────────────────────┘  │
     └─────────────────────────────────────────────┘
     │
     └─────────────────────────────────────────────┐
       页面 locale 辅助流（由 Accept-Language 决定）
        ┌──────────────────────────────────────┐
        │ findMatchingLocale(Accept-Language)  │
        │ → 输出 locale="zh-CN"（如果匹配）    │
        └──────────────────┬───────────────────┘
                           ↓
        ┌──────────────────────────────────────┐
        │ 传递给 <VerifyEmail locale={locale}> │
        │ → 邮件验证页面使用中文               │
        └──────────────────────────────────────┘
```

---

## 七、设计要点与注意事项

### 7.1 核心设计原则

1. **双轨控制**：问卷核心内容由 URL 显式控制（保证分享一致性），周边流程由浏览器语言隐式控制（优化体验）
2. **双轨回退**：问卷内容使用 `default` 键回退（业务可控），UI 使用硬编码 `en` 回退（技术稳定）
3. **双版本实现**：答题端保证可用性（回退 default），管理端保证可见性（不回退，显示空值提示补全）
4. **"default" 不是真实语言**：它是一个指针，指向问卷管理员设置的默认语言，实际渲染时会解析为真实语言代码
5. **字段级独立回退**：每个字段独立判断是否需要回退，避免因单个字段缺译导致整个语言版本不可用

### 7.2 常见陷阱

| 陷阱 | 现象 | 规避方法 |
|-----|------|---------|
| 显式空字符串不回退 | 某语言翻译被设为 `""`，受访者看到空白 | 在管理端验证时，检查启用语言的字段不能为空白 |
| 禁用语言不报错 | `?lang=disabled` 静默回退默认语言 | 分享链接时只提供启用语言的选项 |
| alias 与 code 冲突 | 别名与其他语言 code 相同导致匹配歧义 | 管理端已做验证，禁止设置冲突的别名 |
| **Accept-Language 不影响问卷** | 修改浏览器语言设置，问卷内容语言不变 | 这是**预期行为**，问卷语言仅由 URL 控制 |
| `default` 不是语言代码 | 误以为 `TI18nString` 中有 `"default"` 语言代码 | `default` 是键名，代表问卷默认语言的内容 |
| **两套 getLocalizedValue** | 同样的输入在管理端和答题端输出不同 | 注意两个实现的回退逻辑差异 |

### 7.3 典型场景示例

**场景**：问卷默认语言为英语（`en`），已添加法语（`fr`）但标题翻译为空，德语（`de`）已被管理员禁用。受访者浏览器 Accept-Language 为 `zh-CN`。

| 访问链接 | 问卷内容语言 | 页面标题（metadata） | 邮件验证页面语言 | 说明 |
|---------|-------------|---------------------|-----------------|------|
| `/s/abc?lang=en` | 英语 | 英语标题 | 中文 | 精确匹配 |
| `/s/abc?lang=fr` | **空标题** | **空标题** | 中文 | 法语翻译被显式设为空，不回退 |
| `/s/abc?lang=de` | 英语（default） | 英语标题 | 中文 | 德语被禁用，回退默认语言 |
| `/s/abc?lang=invalid` | 英语（default） | 英语标题 | 中文 | 语言不存在，回退默认 |
| `/s/abc` | 英语（default） | 英语标题 | 中文 | 无 lang 参数，回退默认（**Accept-Language=zh-CN 不影响问卷**） |
