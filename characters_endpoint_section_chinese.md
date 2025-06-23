## 5. 后端角色数据处理 (`src/endpoints/characters.js`)

`src/endpoints/characters.js` 文件是后端处理所有与角色卡相关的CRUD（创建、读取、更新、删除）操作以及导入导出功能的核心。它利用前面章节讨论的解析器、数据结构和验证器，为前端提供了一套完整的API接口。

### 5.1 缓存策略 (Caching Strategy)

为了提升性能并减少对文件系统的频繁访问，该模块实现了两级缓存策略：内存缓存和磁盘缓存。

*   **内存缓存 (`memoryCache`)**:
    *   这是一个 `MemoryLimitedMap` 的实例，用于在应用程序的内存中缓存已解析的角色卡数据（通常是JSON字符串）。
    *   **缓存键 (Cache Key)**: 格式为 `\${filePath}-\${stat.mtimeMs}`，即文件路径加上文件的最后修改时间戳（毫秒）。这意味着当文件内容发生变化时，其 `mtimeMs` 会更新，从而导致缓存键失效，确保读取到的是最新数据。
    *   **容量配置 (`memoryCacheCapacity`)**: 内存缓存的总容量可以通过配置项 `performance.memoryCacheCapacity` (例如设置为 "100mb") 进行限制，防止无限制地消耗内存。在内存受限的环境（如某些Android设备）下，可能会禁用此缓存 (`isAndroid` 变量判断)。

*   **磁盘缓存 (`diskCache`)**:
    *   通过 `DiskCache` 类实现，提供了一个基于文件系统的持久化缓存。缓存文件存储在 `DATA_ROOT/_cache/characters/` 目录下。
    *   **目的**: 即便应用重启，通过磁盘缓存也能快速加载已解析的角色数据，减少首次访问的延迟。
    *   **缓存键**: 与内存缓存使用类似的键生成逻辑。
    *   **同步与清理 (`syncQueue`, `SYNC_INTERVAL`)**:
        *   `DiskCache` 维护一个 `syncQueue` (一个 `Set` 结构)，记录需要同步的用户句柄。
        *   通过 `SYNC_INTERVAL` (默认为5分钟) 定义的定时器，周期性地调用 `#syncCacheEntries` 方法。
        *   该方法会进一步调用 `verify`，`verify` 方法会遍历指定用户目录下的实际角色PNG文件，生成有效的缓存键集合，然后与磁盘缓存目录中的文件进行比对，移除那些在文件系统中已不存在对应源文件的无效缓存条目（例如，角色被删除后）。
    *   **启用配置 (`useDiskCache`)**: 可以通过 `performance.useDiskCache` (默认为 `true`) 配置项启用或禁用磁盘缓存。

*   **统一读取入口 (`readCharacterData` 函数)**:
    *   此函数是读取角色卡数据的统一入口点。
    *   **读取顺序**:
        1.  首先尝试从 `memoryCache` 中获取数据。
        2.  如果内存缓存未命中，且磁盘缓存 (`useDiskCache`) 已启用，则尝试从 `diskCache` 中获取数据。
        3.  如果两级缓存均未命中，则调用 `character-card-parser.js` 中的 `parse` 函数直接从PNG文件解析数据。
    *   **缓存写入**: 当从文件成功解析数据后，会将结果同时存入内存缓存（如果未禁用）和磁盘缓存（如果启用），以便后续快速访问。

### 5.2 核心处理函数 (Core Processing Functions)

*   **`writeCharacterData(inputFile, data, outputFile, request, crop)`**:
    *   **作用**: 负责将字符数据（`data`，一个JSON字符串）写回PNG文件。
    *   **缓存失效**: 在写入前，会尝试清除内存缓存中与 `inputFile` 相关的条目。如果启用了磁盘缓存，会将当前用户的句柄加入 `diskCache.syncQueue`，以便后续校验和清理。
    *   **图像处理**:
        *   如果 `inputFile` 是一个已存在的图片路径或Buffer，它会使用 `Jimp` 库进行图像处理。
        *   可以进行裁剪 (`crop` 参数) 和尺寸调整 (覆盖至 `AVATAR_WIDTH`, `AVATAR_HEIGHT` 定义的标准头像尺寸)。
        *   如果读取或处理输入图片失败，会使用预设的默认头像 (`defaultAvatarPath`)。
    *   **数据嵌入**: 调用 `character-card-parser.js` 中的 `write` 函数，将 `data` 字符串（Base64编码后）嵌入到处理后的图像缓冲区中。
    *   **原子写入**: 使用 `write-file-atomic` 库将最终的PNG缓冲区写入到用户角色目录 (`request.user.directories.characters`) 下的 `outputFile.png`。原子写入能确保文件写入过程的完整性，防止因意外中断导致文件损坏。

*   **`processCharacter(item, directories, { shallow })`**:
    *   **作用**: 读取指定的角色PNG文件 (`item`)，解析其内部数据，并计算一些统计信息。
    *   **数据读取与转换**:
        *   使用 `readCharacterData` (前面讨论的缓存读取函数) 获取原始JSON数据。
        *   调用 `getCharaCardV2` 将解析出的JSON对象转换为内部统一的V2格式。
    *   **统计信息计算**:
        *   `date_added`: 文件创建时间。
        *   `create_date`: 角色创建日期（可能来自卡片数据或文件创建时间）。
        *   `chat_size`, `date_last_chat`: 通过 `calculateChatSize` 遍历角色对应的聊天记录目录 (`directories.chats`) 计算聊天文件总大小和最新聊天时间。
        *   `data_size`: 通过 `calculateDataSize` 估算角色定义数据的大小。
    *   **浅加载 (`shallow` loading)**:
        *   如果 `shallow` 选项为 `true` (通常由 `useShallowCharacters` 配置控制)，则调用 `toShallow(character)` 函数。
        *   `toShallow` 只返回角色对象的一个子集，包含列表视图所必需的关键字段（如 `name`, `avatar`, `fav`, `tags`, 部分 `data` 字段等），以减少传输数据量和前端处理负担。

*   **`charaFormatData(data, directories)`**:
    *   **作用**: 这是一个关键的数据规范化函数，用于将来自不同来源（如前端表单提交、JSON导入）的输入数据 `data` 转换为标准的V2角色卡JSON结构。
    *   **处理逻辑**:
        *   它会尝试从 `data.json_data` (如果存在) 解析一个基础对象，然后用 `data` 中的其他字段覆盖或填充。
        *   为V1和V2规范中的各个字段设置默认值 (例如，空字符串、`0.5` for `talkativeness`, `false` for `fav`)。
        *   进行数据类型转换 (例如，将逗号分隔的标签字符串转换为数组)。
        *   构建嵌套对象，如 `data.extensions` (包含 `talkativeness`, `fav`, `world`, `depth_prompt`) 和 `data.character_book` (如果 `data.world` 存在，会尝试读取对应的世界信息文件并转换为 `character_book` 结构)。
    *   **用途**: 在 `convertToV2` 中被调用，确保从V1转换来的数据以及新创建的数据都符合V2的结构。

*   **版本转换 (`convertToV2`, `readFromV2`)**:
    *   `convertToV2(char, directories)`:
        *   **作用**: 将一个V1格式的角色卡对象 `char` 升级到V2结构。
        *   **逻辑**: 实质上是调用 `charaFormatData`，将V1对象的字段作为输入传递给 `charaFormatData`，从而构建出一个完整的V2结构对象。它负责将旧的字段名（如 `creatorcomment`）映射到新的字段路径（如 `data.creator_notes`）。
    *   `readFromV2(char)`:
        *   **作用**: 处理已经是V2格式（即包含 `spec: 'chara_card_v2'`）的卡片对象。
        *   **逻辑**: 主要确保所有预期的V2字段（特别是 `data.extensions` 中的字段如 `talkativeness`, `fav`）都存在于对象中。如果某些扩展字段缺失，它会尝试用默认值回填这些字段，以保证后续代码处理的健壮性。它还会将 `data` 对象中的某些值同步到顶层对象属性，以兼容一些期望直接访问这些属性的代码。

### 5.3 API 端点功能概述 (API Endpoint Overview)

该文件使用 Express.js 的 `Router` 定义了一系列HTTP API端点，用于管理角色数据。

*   `/create` (POST): 创建新角色。接收表单数据和可选的头像图片上传，使用 `charaFormatData` 处理数据，然后调用 `writeCharacterData` 保存。
*   `/rename` (POST): 重命名角色。更新角色文件名和角色数据内部的名称字段。
*   `/edit` (POST): 编辑现有角色。接收表单数据和可选的新头像，使用 `charaFormatData` 更新数据，调用 `writeCharacterData` 写回。
*   `/edit-attribute` (POST): 编辑角色单个属性。直接修改JSON对象中的指定字段。
*   `/merge-attributes` (POST): 合并多个属性。深度合并请求体中的属性到现有角色数据，并使用 `TavernCardValidator` 验证合并后的结果。
*   `/delete` (POST): 删除角色。删除角色PNG文件，并可选择同时删除其关联的聊天记录。
*   `/all` (POST): 获取所有可用角色的列表。支持通过 `useShallowCharacters` 配置进行浅加载。
*   `/get` (POST): 获取单个角色的详细数据。调用 `processCharacter` 并设置 `shallow: false`。
*   `/import` (POST): 从多种格式导入角色。支持 PNG (直接读取嵌入数据), JSON (V1 或 V2 格式), YAML, 以及 CharX (一种ZIP包格式)。
    *   使用辅助函数如 `importFromPng`, `importFromJson`, `importFromYaml`, `importFromCharX`。
    *   这些函数内部会调用 `readCharacterData` (对于PNG)，或者解析文件内容后调用 `convertToV2` 或 `readFromV2`，最终都通过 `writeCharacterData` 将导入的角色保存为标准的PNG角色卡。
*   `/export` (POST): 导出角色数据。
    *   `png` 格式: 返回带有嵌入式角色数据的PNG文件，私有字段（如 `fav`, `chat`）会被移除。
    *   `json` 格式: 返回V2格式的JSON数据，同样会移除私有字段。
*   `/duplicate` (POST): 复制一个已存在的角色，新角色文件名会自动添加后缀（如 `_1`, `_2`）。

**中间件**:
*   `validateAvatarUrlMiddleware` / `getFileNameValidationFunction`: 用于验证请求中涉及的角色文件名（通常是 `avatar_url` 参数），防止路径遍历等安全问题，确保文件名是合法的。

这些端点共同构成了角色管理的后端服务，为用户界面提供了强大的数据操作能力。
