## 3. 角色卡数据结构定义 (`public/scripts/char-data.js`)

`public/scripts/char-data.js` 文件并不包含直接运行的程序代码，而是主要利用 JSDoc 注释来定义角色卡数据的结构。这些定义作为一种“模式”（schema）或接口规范，为开发者提供了角色数据应包含哪些字段以及这些字段预期类型的清晰指南。

### 3.1 JSDoc Typedefs 的应用

该文件通过 `@typedef` JSDoc 标签来声明一系列自定义的类型，这些类型详细描述了角色卡 (Character Card) 从 V1 到 V2 版本的数据结构，以及其中嵌套的复杂对象，如世界信息 (`WorldInfo`) 和扩展字段 (`Extensions`)。

例如：
```javascript
/**
 * @typedef {object} v2CharData
 * @property {string} name - The character's name.
 * @property {string} description - A brief description of the character.
 * // ... other properties
 */
```
这段 JSDoc 定义了一个名为 `v2CharData` 的对象类型，并指明了它应有的 `name` 和 `description` 属性，两者都应为字符串。这种方式使得在没有 TypeScript 或其他类型系统的情况下，也能对数据结构进行相对严格的定义和文档化。

### 3.2 核心数据结构 `v2CharData` 详解

`v2CharData` 是角色卡V2版本的主要数据结构。以下是其关键属性的解析：

*   `name: string` - **角色名称**: 角色的名字。
*   `description: string` - **角色描述**: 对角色外观、背景故事等的文字描述。
*   `personality: string` - **角色性格**: 通常是一段文本，描述角色的性格特征，有时也可能包含用逗号分隔的关键词。
*   `scenario: string` - **场景设定**: 角色所处的具体情境或世界背景。
*   `first_mes: string` - **首次发言**: 角色在对话开始时主动发出的第一条消息。
*   `mes_example: string` - **对话示例**: 一段或多段示例对话，展示了角色的典型说话风格、语气、用词习惯以及如何使用Markdown进行动作描述（如 `*角色微笑*`）。
*   `creator_notes: string` - **创建者注释**: 创建者留给自己的笔记，例如创作思路、待完善点等，此内容不会直接展示给用户或AI。
*   `tags: string[]` - **标签数组**: 一个字符串数组，包含用于分类和搜索角色的关键词标签 (例如 `["冒险", "幻想", "女性主角"]`)。
*   `system_prompt: string` - **系统提示文本**: 给大型语言模型 (LLM) 的指令，用于引导其在扮演该角色时的行为、语气和回应风格。
*   `post_history_instructions: string` - **对话历史处理指令**: 指导 LLM 如何回顾和利用之前的对话历史来保持上下文连贯性和角色一致性。
*   `creator: string` - **创建者名称**: 制作该角色卡的作者名。
*   `character_version: string` - **角色卡数据版本**: 标识该角色卡数据结构的版本号，例如 "2.0"。
*   `alternate_greetings: string[]` - **其他问候语数组**: 除了 `first_mes` 之外，角色可以使用的其他开场白。

#### 嵌套结构：

*   `character_book: v2WorldInfoBook` - **角色手册/世界信息**: 此字段包含一个 `v2WorldInfoBook` 类型的对象，用于存储与角色相关的结构化世界背景知识。
    *   `v2WorldInfoBook` 结构:
        *   `name: string` - 手册的名称。
        *   `entries: v2DataWorldInfoEntry[]` - 一个 `v2DataWorldInfoEntry` 对象的数组，每个对象代表一条世界信息条目。

*   `extensions: v2CharDataExtensionInfos` - **扩展字段**: 此字段包含一个 `v2CharDataExtensionInfos` 类型的对象，用于存储各种非标准或附加的角色信息。
    *   `v2CharDataExtensionInfos` 结构:
        *   `talkativeness: number` - **健谈度**: 一个数值，可能表示角色的健谈程度。
        *   `fav: boolean` - **收藏状态**: 布尔值，标记该角色是否被用户收藏。
        *   `world: string` - **所属世界**: 角色所属的虚构世界的名称。
        *   `depth_prompt: object` - **深度提问对象**: 包含用于引导更深层次角色互动的提示。
            *   `depth: number` - 提问的深度级别。
            *   `prompt: string` - 实际的提示文本。
            *   `role: "system" | "user" | "assistant"` - 在该深度提问中，角色扮演的身份（系统、用户或助手）。
        *   `regex_scripts: RegexScriptData[]` - 自定义正则表达式脚本数组。
        *   **其他非标准扩展**: 该对象还可以包含由外部工具或特定社区添加的自定义字段，例如：
            *   `pygmalion_id: string` (可选) - Pygmalion.chat 分配的唯一ID。
            *   `chub: object` (可选) - Chub.ai 相关的特定数据。
            *   `risuai: object` (可选) - RisuAI 相关的特定数据。
            *   `sd_character_prompt: object` (可选) - Stable Diffusion相关的角色提示词。
            这体现了数据结构的良好可扩展性。

### 3.3 `v1CharData` 与向后兼容性

文件中也定义了 `v1CharData` 结构。通过对比可以看出 `v2CharData` 是在其基础上发展演变而来的。

*   `v1CharData` 包含了一些基本字段如 `name`, `description`, `personality`, `scenario`, `first_mes`, `mes_example`, `creatorcomment` (对应 V2 的 `creator_notes`), `tags`, `talkativeness`, `fav`, `create_date`。
*   一个显著的区别是，`v1CharData` 中有一个 `data` 字段，其类型被指定为 `v2CharData`。这表明在实际应用中，V1 卡片数据可能被视为一个容器，其核心角色定义已升级到 V2 结构并存储在 `data` 字段内，或者服务器端在加载 V1 卡片时会将其转换为 V2 结构以实现统一处理。
*   例如，在 `src/endpoints/characters.js` (此文件未在此次分析中直接读取，但根据项目结构推断) 中可能存在逻辑，当加载到一个 V1 格式的角色卡时，会将其字段映射或转换为 `v2CharData` 结构，以确保后续处理流程的一致性，从而实现向后兼容。

### 3.4 世界信息条目 (`v2DataWorldInfoEntry`) 结构

`v2DataWorldInfoEntry` 用于定义角色手册 (`character_book`) 中的每一条具体的知识点。

*   `keys: string[]` - **主关键词数组**: 触发此条目信息的主要关键词列表。
*   `secondary_keys: string[]` (可选) - **次要关键词数组**: 辅助触发此条目的关键词列表。
*   `content: string` - **内容**: 该条目的具体信息文本，当关键词被触发时，此内容可能会被注入到LLM的上下文中。
*   `comment: string` - **注释**: 对该条目的人类可读描述或解释。
*   `enabled: boolean` - **启用状态**: 标记此条目当前是否生效。
*   `selective: boolean` - **选择性注入**: 指示此条目的注入是否受特定条件控制。
*   `constant: boolean` - **固定内容**: 指示此条目内容是否固定不变。
*   `insertion_order: number` - **插入顺序**: 定义此条目在满足条件时被注入到上下文中的顺序。
*   `position: string` - **位置**: 指定条目内容在上下文中的应用位置或方式。
*   `id: number` - **唯一ID**: 条目的唯一标识符。
*   `extensions: v2DataWorldInfoEntryExtensionInfos` - **条目扩展字段**: 针对单个世界信息条目的扩展设置。
    *   `probability: number` - 应用此条目的概率 (0到1之间)。
    *   `useProbability: boolean` - 是否启用概率应用。
    *   `scan_depth: number` - 匹配关键词时扫描对话历史的深度。
    *   `case_sensitive: boolean` - 关键词匹配是否区分大小写。
    *   `match_whole_words: boolean` - 是否仅匹配完整单词。
    *   `match_persona_description: boolean` - 是否也针对角色的 `description` 字段进行关键词匹配。
    *   以及其他如 `depth`, `selectiveLogic`, `group`, `prevent_recursion` 等更细致的控制参数。

通过这些详细的 JSDoc 类型定义，项目确保了角色卡数据在不同模块和开发者之间的结构一致性和可理解性。
