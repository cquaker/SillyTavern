## 6. 角色手册/世界信息与角色卡的集成

角色手册（Character Book），在代码层面常通过“世界信息”（World Info）文件来实现，为角色提供了一个结构化的知识背景，使其能够在对话中引用特定的世界观、设定或记忆片段。这种集成主要发生在角色创建或编辑时，将外部定义的世界信息内嵌到角色卡数据中。

### 6.1 世界信息读取 (`src/endpoints/worldinfo.js`)

`src/endpoints/worldinfo.js` 模块负责管理世界信息文件的生命周期，包括创建、读取、编辑、删除和导入。其中，`readWorldInfoFile` 函数是连接外部世界信息文件与角色卡处理逻辑的桥梁。

*   **`readWorldInfoFile(directories, worldInfoName, allowDummy)` 函数**:
    *   **参数**:
        *   `directories`: 一个包含用户特定目录路径的对象 (例如 `request.user.directories`)，其中 `directories.worlds` 指向存储世界信息文件的目录。
        *   `worldInfoName`: (字符串) 要读取的世界信息文件的名称 (不含 `.json` 后缀)。
        *   `allowDummy`: (布尔值) 如果为 `true`，当文件不存在时，函数会返回一个包含空 `entries` 对象的“哑”对象，而不是 `null`。
    *   **功能**:
        1.  **路径构建**: 根据 `directories.worlds` 和 `worldInfoName` 构建完整的文件路径，例如 `user_data/worlds/MyWorld.json`。
        2.  **文件读取与解析**:
            *   使用 Node.js 的 `fs.existsSync` 检查文件是否存在。
            *   如果文件存在，则使用 `fs.readFileSync` 同步读取文件内容 (UTF-8编码)。
            *   将读取到的文本内容通过 `JSON.parse()` 解析为 JavaScript 对象。
        3.  **空处理**:
            *   如果 `worldInfoName` 为空或文件不存在：
                *   若 `allowDummy` 为 `true`，返回 `{ entries: {} }`。
                *   若 `allowDummy` 为 `false` (或未提供)，返回 `null`。
    ```javascript
    // 示例: readWorldInfoFile 的核心逻辑
    export function readWorldInfoFile(directories, worldInfoName, allowDummy) {
        const dummyObject = allowDummy ? { entries: {} } : null; // 准备哑对象

        if (!worldInfoName) { // 如果文件名为空
            return dummyObject;
        }

        const filename = `${worldInfoName}.json`; // 构建完整文件名
        const pathToWorldInfo = path.join(directories.worlds, filename); // 构建完整路径

        if (!fs.existsSync(pathToWorldInfo)) { // 文件不存在
            console.error(`World info file ${filename} doesn't exist.`);
            return dummyObject;
        }

        const worldInfoText = fs.readFileSync(pathToWorldInfo, 'utf8'); // 读取文件
        const worldInfo = JSON.parse(worldInfoText); // 解析JSON
        return worldInfo;
    }
    ```

*   **其他 API 端点**:
    *   `worldinfo.js` 还通过 Express Router 暴露了其他几个 API 端点，如：
        *   `/get` (POST): 获取指定世界信息文件的内容。
        *   `/delete` (POST): 删除指定的世界信息文件。
        *   `/import` (POST): 导入世界信息文件 (通常是 JSON 格式)。
        *   `/edit` (POST): 编辑并保存现有的世界信息文件。
    *   这些端点使得用户可以通过界面管理其世界信息库。

### 6.2 在角色创建/编辑时集成 (`src/endpoints/characters.js` 中的处理)

当用户创建新角色或编辑现有角色，并为其指定了一个“世界”（通常是通过一个下拉菜单选择已存在的 World Info 文件名）时，`src/endpoints/characters.js` 中的 `charaFormatData` 函数会负责将这个世界信息集成到角色卡数据中。

*   **`charaFormatData` 函数中的集成逻辑**:
    1.  **检查 `world` 字段**: 该函数会检查传递给它的 `data` 对象 (通常来自前端的角色编辑表单) 是否包含一个 `world` 字段。这个 `world` 字段的值就是用户选择的世界信息文件的名称。
    2.  **读取世界信息**: 如果 `data.world` 存在且有值，`charaFormatData` 会调用 `readWorldInfoFile(directories, data.world, false)` 来获取该世界信息文件的内容。
        ```javascript
        // 在 charaFormatData 函数内部（概念性代码）
        if (data.world) { // 如果表单数据中指定了 world
            try {
                // 从 worldinfo.js 读取世界文件内容
                const worldInfoFileContent = readWorldInfoFile(directories, data.world, false);

                if (worldInfoFileContent && worldInfoFileContent.entries) {
                    // 将读取到的世界信息条目转换为角色手册格式
                    const characterBookData = convertWorldInfoToCharacterBook(data.world, worldInfoFileContent.entries);
                    _.set(char, 'data.character_book', characterBookData); // 设置到角色卡的 character_book 字段
                }
                // ... 如果 worldInfoFileContent.originalData 存在，则直接使用它
            } catch (error) {
                console.warn(`Failed to read world info file: ${data.world}. Character book will not be available.`, error);
            }
        }
        ```
    3.  **转换与赋值 (`convertWorldInfoToCharacterBook` 助手函数)**:
        *   如果成功读取到世界信息文件内容 (并且它包含 `entries`)，`charaFormatData` 会调用一个名为 `convertWorldInfoToCharacterBook(worldName, entries)` 的内部辅助函数。
        *   **目的**: `convertWorldInfoToCharacterBook` 的主要任务是将从 `.json` 文件中读取的扁平化世界信息条目列表（其结构可能与角色卡内部期望的 `character_book` 结构不同）转换为符合 `v2CharData` 中 `character_book` 字段要求的 `v2WorldInfoBook` 格式。
        *   **`v2WorldInfoBook` 格式**: 要求一个包含 `name` (世界名称) 和 `entries` (一个 `v2DataWorldInfoEntry` 对象数组) 的对象。
        *   **字段映射 (部分示例)**:
            *   世界信息条目的 `uid` (唯一ID) 映射到 `originalEntry.id`。
            *   `key` (主关键词数组) 映射到 `originalEntry.keys`。
            *   `keysecondary` (次要关键词数组) 映射到 `originalEntry.secondary_keys`。
            *   `content` (内容) 映射到 `originalEntry.content`。
            *   `comment` (注释) 映射到 `originalEntry.comment`。
            *   布尔值字段如 `constant`, `selective`, `disable` (取反为 `enabled`) 等直接映射。
            *   `order` (插入顺序) _可能_ 映射到 `originalEntry.insertion_order`。
            *   世界信息条目中可能存在的各种类似扩展的字段 (如 `position`, `excludeRecursion`, `probability`, `scanDepth`, `caseSensitive` 等) 会被收集并映射到 `originalEntry.extensions` 对象内部的对应属性。
        *   转换完成后，`convertWorldInfoToCharacterBook` 返回的 `character_book` 对象被赋值给待格式化的角色对象的 `data.character_book` 路径。

通过这种机制，外部定义和管理的世界信息可以在角色创建或更新时被“快照”并嵌入到角色卡数据中。这意味着每个角色卡都包含其在创建/更新时所关联的世界信息的完整副本，使得角色在后续使用中能够独立地访问这些背景知识，而无需在每次对话时都去动态查询外部文件。这也确保了即使原始的世界信息文件后续被修改或删除，已创建的角色卡的行为和知识背景也能保持一致。
