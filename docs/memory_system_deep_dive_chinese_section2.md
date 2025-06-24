## 2. 对话摘要与长期记忆 (`public/scripts/extensions/memory/index.js`)

该模块 (`public/scripts/extensions/memory/index.js`) 实现了对话的自动摘要功能，并将生成的摘要作为一种“长期记忆”形式注入到后续的AI提示中，以帮助AI在长对话中保持上下文连贯性。

### 2.1 初始化与配置加载 (Initialization and Configuration Loading)

*   **初始化入口**: 脚本的主要逻辑包裹在 `jQuery(async function () { ... })` 中，确保在DOM加载完毕后执行。
*   **设置加载 (`loadSettings()`)**:
    *   此函数负责从 `extension_settings.memory` (一个全局对象，存储所有扩展的配置) 中读取记忆扩展的设置。
    *   如果 `extension_settings.memory` 为空，则使用 `defaultSettings` 对象中的预设值进行初始化。
    *   `defaultSettings` 包含诸如 `memoryFrozen` (是否冻结记忆，停止自动摘要), `source` (摘要服务来源), `prompt` (摘要指令模板), `template` (摘要内容注入模板), `position` (注入位置), `depth` (注入深度) 等一系列配置项。
    *   加载设置后，会更新界面上对应的HTML输入元素（如输入框、复选框、下拉菜单）的值，并触发其 `input` 或 `change` 事件以确保UI同步。
        ```javascript
        // 概念性代码片段 - loadSettings()
        function loadSettings() {
            if (Object.keys(extension_settings.memory).length === 0) { // 如果没有已保存的设置
                Object.assign(extension_settings.memory, defaultSettings); // 使用默认设置
            }
            // 确保所有默认设置中的键都存在于 extension_settings.memory 中
            for (const key of Object.keys(defaultSettings)) {
                if (extension_settings.memory[key] === undefined) {
                    extension_settings.memory[key] = defaultSettings[key];
                }
            }
            // 更新UI元素，例如：
            $('#memory_frozen').prop('checked', extension_settings.memory.memoryFrozen).trigger('input');
            $('#summary_source').val(extension_settings.memory.source).trigger('change');
            // ... 其他UI元素的初始化
        }
        ```
*   **事件监听器绑定**:
    *   在 `addExtensionControls()` (通过 `jQuery(async function () { ... })` 调用) 和 `setupListeners()` (可被多次调用，例如在弹出窗口时) 中，会为各个设置相关的UI元素绑定事件监听器。
    *   例如，`$('#memory_frozen').on('input', onMemoryFrozenInput)` 会监听 “冻结记忆”复选框的改变，调用 `onMemoryFrozenInput` 函数，该函数更新 `extension_settings.memory.memoryFrozen` 的值，并通过 `saveSettingsDebounced()` (一个防抖函数) 保存设置。
    *   类似地，`$('#summary_source').on('change', onSummarySourceChange)` 监听摘要服务来源下拉菜单的改变。

### 2.2 摘要触发机制 (Summary Trigger Mechanism)

*   **核心事件监听**: 摘要过程主要由聊天事件触发。
    *   `eventSource.makeLast(event_types.CHARACTER_MESSAGE_RENDERED, onChatEvent)`: 监听角色消息成功渲染到聊天界面后的事件。`makeLast` 确保此回调在其他同类事件回调之后执行。
    *   其他事件如 `MESSAGE_DELETED`, `MESSAGE_UPDATED`, `MESSAGE_SWIPED` 也会调用 `onChatEvent`。

*   **`onChatEvent()` 函数内部逻辑**:
    1.  **前置检查 (Pre-checks)**:
        *   检查模块是否启用 (例如，如果使用 "extras" API 作为摘要来源，但 `modules` 中不包含 'summarize')。
        *   检查 WebLLM 是否受支持 (如果设置为 WebLLM 来源但浏览器不支持)。
        *   检查AI是否正在流式输出 (`streamingProcessor && !streamingProcessor.isFinished`)。
        *   检查是否已在进行API调用 (`inApiCall`) 或记忆是否已冻结 (`extension_settings.memory.memoryFrozen`)。如果任一条件为真，则提前返回。
    2.  **聊天变化跟踪**:
        *   使用 `lastMessageId` (最后一条消息的ID/索引) 和 `lastMessageHash` (最后一条消息内容的哈希值) 来判断是否有新消息或消息内容是否有变动。如果与上次记录一致，则不执行后续操作。
    3.  **删除或编辑消息处理**:
        *   如果当前聊天长度 `chat.length` 小于 `lastMessageId`，说明有消息被删除。此时会调用 `getLatestMemoryFromChat(chat)` 获取最近的有效摘要并用 `setMemoryContext` 更新上下文。
        *   如果最后一条消息的 `extra.memory` 存在，但其内容哈希与 `lastMessageHash` 不符（说明消息被编辑或重生成），则会删除该消息的 `extra.memory`，避免使用过时的摘要。
    4.  **调用摘要**: 调用 `summarizeChat(context)` 尝试进行摘要。
    5.  **更新状态**: 无论摘要是否成功，最后都会更新 `lastMessageId` 和 `lastMessageHash`。

*   **`getSummaryPromptForNow(context, force)` 深度解析**: 此函数判断当前是否需要生成摘要，并返回摘要指令。
    1.  如果 `promptInterval` (摘要间隔消息数) 为0且非强制 (`force` 为 false)，则直接返回空字符串，不进行摘要。
    2.  等待条件：确保当前没有正在发送消息 (`is_send_press === false`) 或群聊生成中 (`is_group_generating === false`)。
    3.  **计算自上次摘要以来的消息数和词数**:
        *   从聊天记录的末尾向前遍历 (`for (let i = context.chat.length - 1; i >= 0; i--)`)。
        *   直到找到一条包含 `extra.memory` (即上次摘要结果) 的消息为止。
        *   累加在此期间的消息数量 (`messagesSinceLastSummary`) 和词语数量 (`wordsSinceLastSummary`)。
    4.  **满足摘要条件**:
        *   如果 `messagesSinceLastSummary >= extension_settings.memory.promptInterval` (达到设定的消息间隔数)。
        *   或者，如果启用了强制词数 (`extension_settings.memory.promptForceWords > 0`) 并且 `wordsSinceLastSummary >= extension_settings.memory.promptForceWords` (达到设定的词数阈值)。
        *   或者，如果是强制执行 (`force` 为 true)。
    5.  **构建摘要指令**: 如果满足条件，使用 `extension_settings.memory.prompt` (用户配置的摘要提示模板，例如："请将以下对话总结在 {{words}} 字以内...") 和 `extension_settings.memory.promptWords` (期望的摘要字数) 通过 `substituteParamsExtended` 函数构建最终的摘要指令字符串。

### 2.3 摘要内容的构建 (`getRawSummaryPrompt`)

当摘要触发条件满足后，如果采用的 `prompt_builder` 是 `RAW_BLOCKING` 或 `RAW_NON_BLOCKING` (通常用于主LLM或WebLLM作为摘要源)，则会调用 `getRawSummaryPrompt(context, prompt)` 来准备发送给摘要模型的实际文本内容。

*   **目的**: 构建一个包含先前摘要（如果有）和近期对话历史的文本块。
*   **获取先前摘要**: 调用 `getLatestMemoryFromChat(chat)` 从聊天记录中找到最近一次存储的摘要内容 (`latestSummary`)。同时通过 `getIndexOfLatestChatSummary(chat)` 获取该摘要所在消息的索引 (`latestSummaryIndex`)。
*   **构建对话缓冲区 (`chatBuffer`)**:
    *   从 `latestSummaryIndex + 1` 开始遍历到当前聊天末尾（排除最后一条消息，因为它通常是触发本次摘要的消息）。
    *   将每条非系统消息格式化为 "用户名: 消息内容" 的形式，并添加到 `chatBuffer` 数组中。
*   **Token 数量控制与截断**:
    *   在循环中，会不断计算当前构建的完整提示（包括摘要指令 `prompt`、先前摘要 `latestSummary` 和当前的 `chatBuffer`）的总 token 数。
        ```javascript
        // 概念性代码片段 - getRawSummaryPrompt 内部的token控制
        // const PADDING = 64; // 为摘要结果本身预留的 token
        // const PROMPT_SIZE = await getSourceContextSize(); // 获取当前摘要源的最大可用上下文大小
        // for (let index = latestSummaryIndex + 1; index < chat.length; index++) {
        //     // ... 添加消息到 chatBuffer ...
        //     const currentFullPrompt = getMemoryString(true); // getMemoryString 内部会组合 prompt, latestSummary, chatBuffer
        //     const tokens = await countSourceTokens(currentFullPrompt, PADDING);
        //     if (tokens > PROMPT_SIZE) {
        //         chatBuffer.pop(); // 如果超出，则移除最后一条消息
        //         break;
        //     }
        //     latestUsedMessage = message; // 记录最后实际使用的消息
        //     // ... maxMessagesPerRequest 控制 ...
        // }
        ```
    *   `countSourceTokens(text, padding)`: 根据 `extension_settings.memory.source` 的设置，调用相应的方法计算 token 数：
        *   `webllm`: 使用 `countWebLlmTokens(text)`。
        *   `extras`: 使用 `getTextTokens(tokenizers.GPT2, text).length` (GPT-2分词器)。
        *   `main`: 使用 `getTokenCountAsync(text, padding)` (主LLM的分词器)。
    *   `getSourceContextSize()`: 计算摘要服务可用的上下文长度。会考虑用户设置的 `overrideResponseLength` (期望的摘要回复长度)，从总上下文长度中减去这部分。
*   **最大消息数限制**: 如果设置了 `extension_settings.memory.maxMessagesPerRequest`，当 `chatBuffer` 中的消息数量达到此限制时，也会停止添加更多消息。
*   **返回**: 函数返回一个对象，包含 `rawPrompt` (组合了 `latestSummary` 和 `chatBuffer` 的最终文本内容) 和 `lastUsedIndex` (最后一条被包含进 `rawPrompt` 的原始消息在聊天记录中的索引)。

### 2.4 与不同摘要后端服务的交互

`summarizeChat(context)` 函数根据 `extension_settings.memory.source` 的设置，将摘要任务路由到不同的处理函数：

*   **`summarizeChatMain(context, force, skipWIAN)`** (用于主LLM源):
    *   首先调用 `getSummaryPromptForNow()` 获取摘要指令。
    *   如果 `extension_settings.memory.prompt_builder` 是 `prompt_builders.DEFAULT`:
        *   调用 `generateQuietPrompt(prompt, false, skipWIAN, '', '', overrideResponseLength)`。这是一个通用的、可能会在后台执行的提示生成函数，`skipWIAN` (Skip World Info And Notes) 控制是否跳过世界信息和作者笔记的注入。
    *   如果 `prompt_builder` 是 `RAW_BLOCKING` 或 `RAW_NON_BLOCKING`:
        *   调用 `getRawSummaryPrompt()` 获取待摘要的文本块 `rawPrompt` 和 `lastUsedIndex`。
        *   调用 `generateRaw(rawPrompt, '', false, false, prompt, overrideResponseLength)` 将原始文本和摘要指令发送给主LLM进行处理。
        *   `RAW_BLOCKING` 会在调用期间禁用发送按钮 (`deactivateSendButtons()`)，并在结束后恢复 (`activateSendButtons()`)。
    *   对LLM返回的结果使用 `removeReasoningFromString()` 清理可能存在的推理步骤或前缀。
    *   如果成功获得摘要，并且上下文未改变 (`!isContextChanged(context)`)，则调用 `setMemoryContext()` 应用摘要。

*   **`summarizeChatWebLLM(context, force)`** (用于WebLLM源):
    *   类似地，先获取摘要指令 `prompt` 和原始文本块 `rawPrompt`。
    *   构建一个消息数组 `messages = [{ role: 'system', content: prompt }, { role: 'user', content: rawPrompt }]`。
    *   调用 `generateWebLlmChatPrompt(messages, params)` 与客户端LLM交互。`params` 中可能会设置 `max_tokens` (基于 `overrideResponseLength`)。
    *   成功后调用 `setMemoryContext()`。

*   **`summarizeChatExtras(context)`** (用于外部Extras API源):
    *   构建 `memoryBuffer` (近期对话) 和 `longMemory` (上次摘要)。
    *   组合成 `resultingString`，并检查其 token 数是否足够进行摘要（避免过短文本）。
    *   调用 `callExtrasSummarizeAPI(resultingString)`。

*   **`callExtrasSummarizeAPI(text)`**:
    *   构造指向 Extras API 的 `/api/summarize` 端点的URL。
    *   使用 `doExtrasFetch` 发送POST请求，请求体中包含待摘要的 `text`。
    *   处理返回的JSON，提取 `summary` 字段。

### 2.5 摘要的注入与应用 (`setMemoryContext`)

当从任一后端成功获取到摘要结果 `value` 后，`setMemoryContext(value, saveToMessage, index = null)` 函数负责将其应用到系统中：

1.  **格式化摘要 (`formatMemoryValue(value)`)**:
    *   使用 `extension_settings.memory.template` (默认是 `[Summary: {{summary}}]`) 和 `substituteParamsExtended` 函数，将原始摘要文本 `value` 包装成最终注入到提示中的格式。
2.  **注册扩展提示 (`setExtensionPrompt`)**:
    *   这是核心步骤，调用 `setExtensionPrompt(MODULE_NAME, formattedValue, position, depth, scan, role)`。
    *   `MODULE_NAME` 被设为 `'1_memory'`，作为此摘要提示片段的唯一标识符。
    *   `formattedValue` 是格式化后的摘要。
    *   `position`, `depth`, `scan`, `role` 均来自用户在记忆扩展设置中的配置。这些参数会告知 `script.js` 中的核心提示构建逻辑，应如何以及在何处将这个摘要片段插入到最终发送给AI的完整提示中。
3.  **更新UI**: 将原始摘要 `value` 设置到界面上的 `#memory_contents` 文本区域，供用户查看和编辑。
4.  **保存到聊天记录 (`saveToMessage`)**:
    *   如果 `saveToMessage` 为 `true`，并且聊天记录存在：
        *   确定要保存摘要的消息索引 `idx`。通常是 `context.chat.length - 2` (即触发本次摘要的消息的前一条消息)，或者是从 `getRawSummaryPrompt` 返回的 `lastUsedIndex` (如果使用RAW模式)。
        *   在对应消息的 `extra` 对象中，将摘要文本 `value` 存储到 `extra.memory` 字段。
        *   调用 `saveChatDebounced()` (防抖函数) 将更新后的聊天记录保存到存储中。

### 2.6 状态管理与用户控制

*   **并发控制 (`inApiCall`)**: 一个布尔标志，在开始调用任何摘要API前设为 `true`，结束后设为 `false`。`onChatEvent` 会检查此标志，防止在已有摘要任务进行时重复触发。
*   **冻结记忆**: `extension_settings.memory.memoryFrozen` 在 `onChatEvent` 中检查，如果为 `true`，则不进行自动摘要。
*   **上下文变化检测 (`isContextChanged(context)`)**: 在异步的摘要API调用返回结果后，会再次检查当前的应用上下文（角色、聊天ID等）是否与发起请求时一致。如果不一致（例如用户切换了角色），则丢弃获取到的摘要结果。
*   **用户交互**:
    *   `forceSummarizeChat(quiet)`: 允许用户手动触发一次摘要。可以通过界面按钮或 `/summarize` 斜杠命令调用。`quiet` 参数控制是否显示提示信息。
    *   `/summarize [text_to_summarize] [--source <src>] [--prompt <p>] [--quiet <q>]`: 斜杠命令，提供更灵活的即时摘要能力，可以指定文本、摘要源和自定义提示。
    *   `onMemoryContentInput()`: 当用户直接编辑 `#memory_contents` 文本区时触发，调用 `setMemoryContext` 更新当前应用的摘要（但不保存到消息的 `extra.memory`）。
    *   `onMemoryRestoreClick()`: 用户点击“恢复”按钮时，会找到存储在消息中的上一个摘要，并用其覆盖当前 `#memory_contents` 的内容，实际上是撤销了手动编辑或清除了当前摘要。

通过这些机制，记忆扩展模块实现了在对话过程中自动或手动生成摘要，并将其作为上下文的一部分提供给AI，从而有效地扩展了AI的“记忆”长度。用户可以通过丰富的设置项和交互操作来控制摘要的行为。
