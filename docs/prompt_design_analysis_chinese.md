# AI角色提示工程深度解析

## 1. 引言

在与AI角色进行交互时，用户体验的自然度、角色行为的一致性以及对话的沉浸感，在很大程度上取决于发送给大型语言模型（LLM）的提示（Prompt）的质量。提示不仅包含了用户的直接输入，更重要的是，它承载了关于角色定义、场景设定、对话历史、以及特定行为指令等一系列复杂信息。一个精心设计的提示能够有效地引导LLM生成符合预期的、具有角色特色且上下文连贯的回应。

本文档旨在从代码层面深入分析本项目中AI角色提示的构建机制与设计理念。我们将详细探讨构成最终提示的各种文本元素来源，剖析系统如何通过预设模板和动态逻辑将这些元素组合成一个完整的上下文，并最终如何针对不同的LLM API进行格式化。通过这一过程，期望能为开发者理解当前提示工程的运作方式、评估其优缺点，并为未来的优化和扩展提供参考。

## 2. 提示词的组成元素 (Prompt Components)

最终发送给LLM的提示是一个精心组合的文本序列，其内容来源于系统中的多个不同模块和数据源。理解这些组成元素是分析提示构建流程的基础。主要元素包括：

*   **角色卡核心字段**:
    *   `description` (角色描述): 角色的外观、背景故事等。
    *   `personality` (角色性格): 定义角色个性的关键词或短语。
    *   `scenario` (场景设定): 角色当前所处的情境。
    *   `first_mes` (首次发言): 角色的开场白，用于新对话的起始。
    *   `mes_example` (对话示例): 展示角色典型对话风格和行为的例子。
    *   `system_prompt` (系统提示): 对LLM的全局指令，如角色扮演的总纲、禁止行为等。
    *   `post_history_instructions` (历史对话处理指令): 指导LLM如何回顾和利用对话历史的特定指令。
    *   `alternate_greetings` (其他问候语): `first_mes`之外的备选开场白。

*   **`character_book` / 世界信息 (World Info)**:
    *   当对话内容触发了角色手册中某个条目的关键词时，该条目的 `content` (具体信息文本) 会被动态注入到提示中。

*   **记忆扩展 (Memory Extension)**:
    *   **对话摘要**: 由记忆扩展生成的对话摘要，通常以特定格式（如 `[Summary: ...]`) 存在，代表了对早期对话内容的概括。
    *   **摘要指令提示**: 记忆扩展在请求LLM（或外部服务）生成摘要时，自身也会使用一个指令性提示（例如：“请将以下对话总结在X字以内...”）。

*   **聊天记录 (Chat History)**:
    *   用户与AI之间最近的几轮对话，经过格式化处理（例如，标记用户和AI的发言者）。

*   **用户当前输入 (User's Current Input)**:
    *   用户最新发送的、等待AI回应的消息。

*   **预设模板自身文本**:
    *   **上下文模板 (`context` presets)**: 例如 `Default.json` 中的 `story_string`，它本身包含了一些固定的引导性文本和用于插入其他动态内容的占位符。
    *   **指令模板 (`instruct` presets)**: 例如 `Alpaca.json` 中的 `input_sequence` (`### Input:\n`) 和 `output_sequence` (`### Response:\n`)，这些是用于标记不同部分的序列标记。还包括 `stop_sequence` (停止序列)。

*   **动态生成的名称**:
    *   `{{char}}`: 代表AI角色的名字。
    *   `{{user}}`: 代表用户的名字。

*   **其他扩展注入的文本**:
    *   例如“作者笔记”（Author's Notes / Floating Prompts）扩展，允许用户在主提示之外动态注入额外的上下文或指令。
    *   其他任何通过 `setExtensionPrompt()` 机制注册的扩展都可能向提示中添加内容。

这些元素共同构成了传递给LLM的完整信息上下文，指导其生成回应。

## 3. 提示词构建流程详解 (Detailed Prompt Construction Process)

提示的构建是一个多阶段的过程，它始于预设的模板，然后通过前端JavaScript逻辑动态地填充和组合来自不同来源的数据，最后再由后端针对特定LLM进行最终格式化。

### 3.1 预设模板：提示构建的蓝图 (`default/content/presets/`)

预设模板定义了提示的基本结构和指令风格。它们分为两类：

*   **上下文模板 (`context` presets)**: 例如 `Default.json`。
    *   核心是 `story_string` 字段，它是一个包含了多种占位符的字符串模板。这些占位符（如 `{{system}}`, `{{description}}`, `{{personality}}`, `{{scenario}}`, `{{first_mes}}`, `{{mes_example}}`, `{{wiBefore}}`, `{{wiAfter}}`, `{{memory}}`, `{{authorsnote}}`, `{{persona_description}}`, `{{char}}`, `{{user}}` 等）会在运行时被替换为实际的角色数据、世界信息、记忆摘要等。
    *   `story_string` 的设计直接决定了角色定义信息在最终提示中的组织方式和呈现顺序。例如，`Default.json` 的 `story_string` 将系统提示、角色描述、性格、场景等信息置于对话历史之前。
    *   不同的上下文模板可以通过调整 `story_string` 中占位符的顺序和周围的引导文本，来实现不同的提示风格（例如，更侧重故事性、角色扮演或纯粹问答）。

*   **指令模板 (`instruct` presets)**: 例如 `Alpaca.json`。
    *   主要定义了在对话历史和用户当前输入部分如何标记不同的发言轮次和角色。
    *   `turn_template`: 定义单轮对话的格式，通常包含 `{{role}}` 和 `{{message}}` 占位符。
    *   `role_map`: 将内部的角色标识（如 `user`, `char`, `system`）映射到模板中实际使用的发言者名称（例如，Alpaca中用户是 "Human"，AI是 "Assistant"）。
    *   `input_sequence` / `output_sequence`: 在每轮用户输入和AI输出之前添加的前缀/序列标记（例如，`### Input:`）。
    *   `stop_sequence`: LLM生成内容时应停止的特殊标记。
    *   `history_sequence`: （如果存在）用于标记整个对话历史部分的开始。
    *   `name_strip_regex`: 用于从角色发言中移除可能存在的前缀（如角色名）的正则表达式。
    *   `chat_start_template`: （如果存在）用于在对话历史开始前添加的模板。

这些模板为提示的自动化构建提供了灵活的框架。用户可以通过选择或自定义这些预设来调整AI的行为。

### 3.2 前端动态组合逻辑 (`public/script.js` - 主要在 `Generate()` 函数中)

前端的 `public/script.js` 文件（特别是其 `Generate()` 函数及其调用的辅助函数）是动态构建提示内容的核心。当用户发送消息或系统需要生成AI回复时，会执行以下主要步骤：

1.  **获取上下文数据 (`CONTEXT`)**:
    *   从当前应用状态中获取角色对象 (`character`)、聊天记录 (`chat`)、用户设置等。

2._  **收集角色卡字段和扩展内容**:
    *   `getCharacterCardFields(char)`: 从角色对象中提取 `description`, `personality`, `scenario`, `first_mes`, `mes_example`, `system_prompt`, `post_history_instructions` 等字段。
    *   `getWorldInfoPrompt()`: （推测）处理 `character_book` 中的世界信息，根据触发条件（本轮分析未深入此动态触发逻辑）生成 `wiBefore` 和 `wiAfter` 的内容。
    *   `getExtensionPrompt(extension_prompt_types.IN_PROMPT, extension_prompt_roles.SYSTEM)` 等调用: 获取由各扩展（如记忆扩展的摘要、作者笔记等）通过 `setExtensionPrompt()` 注册并希望注入到主提示中的内容。这些内容会根据其注册时指定的 `position` (例如，`IN_SYSTEM_PROMPT`, `BEFORE_CHAT`, `AFTER_CHAT`) 和 `role` 被收集起来。

3.  **渲染上下文模板 (`renderStoryString()`)**:
    *   调用 `renderStoryString(CONTEXT.story_string, contextArgs)`，其中 `contextArgs` 是一个包含了上一步收集到的所有动态内容（角色字段、世界信息、扩展提示等）的对象。
    *   `renderStoryString` 内部使用 `substituteParams()` (或类似宏替换逻辑) 将 `story_string` (来自选定的上下文预设) 中的占位符 (如 `{{system}}`, `{{memory}}`) 替换为 `contextArgs` 中对应的值。
    *   这一步生成了提示中非对话历史的部分，即“故事背景”或“系统设定”部分。

4.  **格式化聊天记录**:
    *   遍历 `chat` 数组中的消息。
    *   使用 `formatMessageHistoryItem(message, instructPreset)` 对每条消息进行格式化。此函数会根据选定的指令预设 (`instructPreset`)：
        *   使用 `instructPreset.role_map` 转换发言者角色。
        *   应用 `instructPreset.turn_template`。
        *   添加 `input_sequence` 和 `output_sequence`。
    *   `doChatInject(message)`: 在格式化每条消息后，调用此函数处理希望在特定聊天消息旁注入的扩展内容 (`extension_prompt_types.IN_CHAT`)。
    *   格式化后的聊天记录字符串会被累加起来。

5.  **组合与截断 (`getCombinedPrompt()`, `checkPromptSize()`)**:
    *   `getCombinedPrompt(storyString, chatString, userInput)`: 将渲染好的 `storyString`、格式化的 `chatString` 和用户当前的 `userInput` 组合成一个初步的完整提示。
    *   `checkPromptSize(prompt, contextTokens, instructTokens)`: 检查组合后的提示是否超出了LLM的上下文长度限制。如果超出，会尝试通过截断聊天记录（从最旧的开始）来缩减提示长度，直到其符合限制。这个过程是迭代的，确保最重要的近期信息被保留。

6.  **针对OpenAI的特殊处理 (`prepareOpenAIMessages()`)**:
    *   如果目标是OpenAI的聊天模型 (Chat Completion API)，系统不会简单地拼接成一个大字符串，而是调用 `prepareOpenAIMessages()`。
    *   此函数将 `storyString` (通常作为 `role: 'system'` 的第一条消息)、格式化后的聊天记录中的每条消息，以及用户输入，分别构造成符合OpenAI API要求的消息对象数组 (例如 `[{role: 'system', content: '...'}, {role: 'user', content: '...'}, ...]` )。
    *   它也会处理 `name` 字段（用于标记发言者，特别是多角色场景）。

最终，经过这些步骤处理后的提示（字符串或消息对象数组）会被发送到后端，再由 `src/prompt-converters.js` 进行最后的适配。

### 3.3 宏替换机制 (`substituteParams`, `evaluateMacros`)

在提示构建的多个阶段，系统广泛使用宏替换机制来动态插入特定值：

*   **`substituteParams(str, data, options)` / `substituteParamsExtended(str, data, options)`**:
    *   这些函数负责替换字符串 `str` 中的 `{{placeholder}}` 形式的占位符。
    *   `data` 对象提供了占位符名称到实际值的映射。例如，`data = { char: '角色名', user: '用户名' }` 会将 `{{char}}` 替换为 "角色名"。
    *   这主要用于替换上下文模板 (`story_string`) 中的角色、用户名称，以及注入角色卡字段、记忆摘要等。

*   **`evaluateMacros(text, context)`**:
    *   （推测，基于 `MacrosParser.registerMacro('summary', ...)` 的存在）系统可能还支持更复杂的宏，例如 `{{MACRO_NAME(arg1, arg2)}}` 或者像 `{{summary}}` 这样直接调用一个已注册宏的函数来动态生成内容。
    *   `MacrosParser` 允许注册自定义宏，这些宏在求值时可以执行JavaScript函数并返回结果字符串，用于替换宏本身。例如，`{{summary}}` 宏在 `memory/index.js` 中注册，用于获取最新的记忆摘要。

这些宏替换机制使得提示模板可以非常灵活，并且能够动态地适应不同的角色、用户和对话状态。

## 4. 针对不同 LLM 的最终格式化 (`src/prompt-converters.js`)

前端构建的初步提示（通常是一个 `messages` 对象数组，其中每条消息包含 `role` 和 `content`，以及可选的 `name`）在发送到后端后，会经过 `src/prompt-converters.js` 模块的处理，以适配目标LLM的特定API格式。

*   **职责**: 该模块的核心职责是将统一的内部消息格式转换为各个LLM（如Claude, Cohere, Google Gemini, OpenAI GPT等）的专有请求结构。
*   **主要转换逻辑**:
    *   **角色映射**: 将内部的角色标识（`system`, `user`, `assistant`, `tool`）映射到目标LLM API要求的角色名称（例如，Claude中可能是 `user`, `assistant`；Google中是 `user`, `model`）。
    *   **系统提示处理**:
        *   某些LLM（如Claude v2.1+, Google Gemini）有专门的 `system` 参数或 `system_instruction` 字段来接收系统级指令。转换器会将输入 `messages` 数组中开头的 `role: 'system'` 消息内容提取出来，放到这个专用字段中。
        *   对于不支持独立系统提示参数的LLM，系统提示内容可能会被格式化为第一条用户消息，或者与其他用户消息合并。
    *   **消息合并与排序**:
        *   一些LLM要求严格的轮替角色（如用户-助手-用户-助手...）。如果输入 `messages` 数组中存在连续相同角色的消息，转换器（如 `mergeMessages` 或特定LLM的转换函数）会尝试将它们合并成一条消息，或者根据需要插入占位消息。
        *   处理图片等多模态内容时，可能会调整消息顺序以符合API要求（例如，Claude要求图片后必须有用户文本）。
    *   **特殊内容类型转换**: 将内部表示的多模态内容（如图片URL）转换为目标LLM API要求的格式（例如，Base64编码的图片数据和对应的MIME类型）。工具调用（Tool/Function Calling）的相关字段也会被转换。
    *   **名称处理**: 对于支持消息 `name` 属性的LLM，会保留或转换该属性；对于不支持的，可能会将名称预置到消息内容中。
    *   **特定API参数**: 除了消息内容本身，转换器可能还会根据需要准备或调整其他API参数，如温度、停止序列等（尽管这些参数的设置主要在后端API配置层面）。

通过这种方式，`prompt-converters.js` 确保了无论前端如何组装通用格式的提示信息，最终发送给不同LLM的都是其能够正确理解和处理的、高度优化的请求。这使得系统能够灵活地接入和切换多种LLM后端，而无需大幅修改核心的提示构建逻辑。

## 5. 示例分析 (Example Analysis)

让我们以一个简化的例子，结合 `Default.json` 上下文预设，来大致演示提示的构建过程。

**假设我们有以下数据**:

*   **角色卡 (Character Card)**:
    *   `name` (`{{char}}`): "艾拉"
    *   `description`: "一位友善的AI助手。"
    *   `system_prompt`: "请始终保持礼貌和乐于助人。"
    *   `personality`: "耐心，细致"
    *   `scenario`: "在数字空间中与用户对话。"
*   **世界信息 (World Info / `character_book`)**:
    *   一个被触发的条目 `content` (`{{wiBefore}}` 或 `{{wiAfter}}`): "[艾拉当前正在处理一项复杂的计算任务。]"
*   **记忆扩展 (Memory Extension)**:
    *   生成的摘要 (`{{memory}}`): "[用户询问了关于天气的问题，艾拉给出了回答。]"
*   **用户 (`{{user}}`)**: "用户A"

**`Default.json` 的 `story_string` (简化版，仅保留部分占位符)**:
```
{{system}}
{{char}}'s Persona: {{description}}
Personality: {{personality}}
Scenario: {{scenario}}
{{wiBefore}}
[Memory: {{memory}}]
{{wiAfter}}
```

**构建过程 (前端 `script.js` 层面，简化)**:

1.  **收集 `contextArgs`**:
    ```javascript
    const contextArgs = {
        system: "请始终保持礼貌和乐于助人。", // 来自 character.system_prompt
        char: "艾拉",
        user: "用户A",
        description: "一位友善的AI助手。",
        personality: "耐心，细致",
        scenario: "在数字空间中与用户对话。",
        wiBefore: "[艾拉当前正在处理一项复杂的计算任务。]", // 假设此WI条目位置为 wiBefore
        memory: "[用户询问了关于天气的问题，艾拉给出了回答。]", // 来自记忆扩展
        wiAfter: "" // 假设没有 wiAfter 内容
        // ... 其他如 first_mes, mes_example 等根据情况填充
    };
    ```

2.  **渲染 `story_string` (通过 `renderStoryString`)**:
    将 `contextArgs` 的值替换到 `story_string` 的占位符中，得到 `storyString`:
    ```
    请始终保持礼貌和乐于助人。
    艾拉's Persona: 一位友善的AI助手。
    Personality: 耐心，细致
    Scenario: 在数字空间中与用户对话。
    [艾拉当前正在处理一项复杂的计算任务。]
    [Memory: [用户询问了关于天气的问题，艾拉给出了回答。]]
    ```
    （注意：实际的 `[Memory: ...]` 格式化由记忆扩展的 `template` 设置决定）

3.  **格式化聊天记录 (假设有以下对话历史，并使用类似Alpaca的指令预设)**:
    *   用户A: "你好，艾拉。"
    *   艾拉: "你好，用户A！有什么可以帮助你的吗？"

    使用 `formatMessageHistoryItem` 和 `instruct` 预设（例如，`input_sequence: "### Human:\n"`, `output_sequence: "### Assistant:\n"`）处理后，得到的 `chatString` 可能如下:
    ```
    ### Human:
    你好，艾拉。
    ### Assistant:
    你好，用户A！有什么可以帮助你的吗？
    ```

4.  **用户当前输入**: 假设用户最新输入是 "今天天气怎么样？"

5.  **组合初步提示 (`getCombinedPrompt`)**:
    将 `storyString`, `chatString`, 和用户当前输入组合起来（这里简单拼接，实际还会添加指令预设的序列标记）：
    ```
    请始终保持礼貌和乐于助人。
    艾拉's Persona: 一位友善的AI助手。
    Personality: 耐心，细致
    Scenario: 在数字空间中与用户对话。
    [艾拉当前正在处理一项复杂的计算任务。]
    [Memory: [用户询问了关于天气的问题，艾拉给出了回答。]]

    ### Human:
    你好，艾拉。
    ### Assistant:
    你好，用户A！有什么可以帮助你的吗？
    ### Human:
    今天天气怎么样？
    ### Assistant:
    ```
    （末尾的 `### Assistant:` 是指令预设添加的，提示LLM开始生成AI的回复。）

这个组合后的文本（或者如果是OpenAI Chat格式，则是一个消息对象数组）随后会被发送到后端，再由 `src/prompt-converters.js` 根据目标LLM的要求进行最终的格式调整（例如，角色名称的转换，系统提示的提取等）。

这个示例简化了许多细节（如占位符的确切名称、所有扩展内容的处理、上下文长度管理等），但它清晰地展示了如何从多个来源收集信息，通过模板和动态替换，逐步构建出一个结构化的、信息丰富的提示，以引导LLM产生期望的输出。

## 6. 总结

本项目中的AI角色提示工程展现了一个高度灵活、可配置且模块化的设计策略，旨在为多样化的LLM后端提供精确且富有上下文的输入。

**设计特点与优势**:

*   **灵活性与可配置性**: 通过引入上下文预设（`context` presets）和指令预设（`instruct` presets），系统允许用户或开发者通过JSON配置文件轻松定制提示的基本结构、角色称呼、序列标记乃至整体的交互风格。占位符（如 `{{system}}`, `{{char}}`, `{{memory}}`）和宏替换机制 (`substituteParams`, `evaluateMacros`) 的广泛应用，使得动态内容能够无缝融入这些预设模板中。
*   **模块化**: 提示的构建流程被清晰地划分为几个阶段：数据收集（角色卡字段、世界信息、记忆摘要、扩展内容）、基于上下文模板的初步渲染、聊天记录的格式化、以及最终由 `prompt-converters.js` 执行的LLM特定格式化。这种分离使得每一部分的逻辑都相对独立，易于理解和维护。例如，添加对新LLM的支持主要集中在 `prompt-converters.js` 中实现新的转换函数，而核心的提示内容构建逻辑则保持不变。
*   **全面的上下文信息**: 系统精心设计，能够将来自角色卡定义、动态生成的世界信息、对话摘要（长期记忆）、实时聊天记录以及用户当前输入等多种来源的信息整合到提示中，为LLM提供了丰富的上下文。
*   **对扩展友好**: 通过 `setExtensionPrompt()` 等机制，允许其他扩展模块（如作者笔记、自定义指令等）在提示构建过程中的特定位置注入内容，进一步增强了提示的定制能力。

**对角色行为与系统功能的影响**:

*   **角色一致性**: 通过在提示中稳定地包含角色描述、性格、场景设定以及系统指令（包括 `post_history_instructions`），有助于LLM在对话过程中保持角色的一致性。
*   **多LLM兼容性**: `prompt-converters.js` 的存在是实现多LLM后端支持的关键。它将内部统一的提示结构转换为各个LLM的特定格式，使得系统核心逻辑不必关心底层LLM的差异。
*   **上下文长度管理**: 前端 `checkPromptSize()` 函数通过动态截断聊天记录，确保了即使在信息源众多的情况下，最终提示也能符合目标LLM的上下文窗口限制。

**潜在复杂性与优化方向**:

*   **调试难度**: 由于提示是由多个模块、多个数据源、多层模板和替换逻辑动态构建起来的，当出现非预期的LLM行为时，追踪和调试具体是哪部分提示内容导致的问题可能会比较复杂。提供更完善的“最终提示预览”或“构建步骤分解”功能可能有助于排查。
*   **性能**: 尽管有缓存机制（如记忆摘要的缓存），但在每次生成回复时，前端都需要执行密集的字符串操作和可能的DOM查询（如果扩展内容涉及UI）。对于非常长的对话历史或极其复杂的提示结构，前端的性能表现可能需要关注。
*   **模板与逻辑的耦合**: 虽然预设提供了灵活性，但某些复杂的提示逻辑（例如，条件性地包含某个元素）可能难以仅通过模板占位符表达，而不得不在JavaScript代码中实现，这可能导致模板配置与前端代码之间存在一定的耦合。

总体而言，该项目的提示设计与构建系统是一个强大而精密的工程实践。它成功地平衡了灵活性、功能丰富性和对多LLM的适应能力，为塑造生动、一致且能进行深度交互的AI角色奠定了坚实的基础。
