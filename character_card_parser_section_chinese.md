## 2. 角色卡数据的封装与解析 (`src/character-card-parser.js`)

`src/character-card-parser.js` 模块是处理角色卡数据与 PNG 文件之间转换的核心。它负责将角色定义（一个 JSON 字符串）写入 PNG 文件的元数据中，以及从 PNG 文件中读取这些数据。

### 2.1 PNG tEXt 区块与 Base64 编码

角色卡数据本质上是一个 JSON 格式的字符串。为了将其嵌入到 PNG 文件中，该 JSON 字符串会首先进行 **Base64 编码**。编码后的字符串随后被存储在 PNG 文件的 `tEXt` 文本区块（chunks）中。

为了标识这些存储角色数据的特定 `tEXt` 区块，解析器使用了特定的关键词 (keyword)：
*   `chara`: 用于标识符合 V2 (版本2) 角色卡规范的数据。
*   `ccv3`: 用于标识符合 V3 (版本3) 角色卡规范的数据。

这种机制允许在一张图片中同时嵌入不同版本的角色卡数据，并能通过关键词区分它们。

### 2.2 写入角色数据 (`write` 函数详解)

`write` 函数负责将角色数据（`data` 参数，一个 V2 格式的 JSON 字符串）写入提供的 PNG 图片缓冲区（`image` 参数）。其主要依赖库包括 `png-chunks-extract` 用于提取现有 PNG 区块，以及 `png-chunk-text` 用于编码和解码 `tEXt` 区块。

其工作流程如下：

1.  **提取现有区块**:
    ```javascript
    // const extract = require('png-chunks-extract'); // CommonJS 风格，实际代码为 ES Module import
    const chunks = extract(new Uint8Array(image)); // 从输入图像中提取所有 PNG 区块
    ```

2.  **移除旧的角色数据区块**: 为了避免数据冗余或冲突，函数会遍历所有 `tEXt` 区块，并移除任何已存在的 `chara` 或 `ccv3` 区块。
    ```javascript
    // const PNGtext = require('png-chunk-text'); // CommonJS 风格
    const tEXtChunks = chunks.filter(chunk => chunk.name === 'tEXt');
    for (const tEXtChunk of tEXtChunks) {
        const decodedChunk = PNGtext.decode(tEXtChunk.data); // 解码 tEXt 区块内容
        if (decodedChunk.keyword.toLowerCase() === 'chara' || decodedChunk.keyword.toLowerCase() === 'ccv3') {
            chunks.splice(chunks.indexOf(tEXtChunk), 1); // 移除匹配的旧区块
        }
    }
    ```

3.  **添加新的 V2 (`chara`) 区块**:
    *   输入的 `data` (V2 JSON 字符串) 进行 UTF-8编码，然后转为 Base64 字符串。
    *   使用 `PNGtext.encode('chara', base64EncodedData)` 创建新的 `tEXt` 区块。
    *   该新区块被插入到 PNG 区块列表的末尾（但在 `IEND` 区块之前）。
    ```javascript
    const base64EncodedDataV2 = Buffer.from(data, 'utf8').toString('base64');
    chunks.splice(-1, 0, PNGtext.encode('chara', base64EncodedDataV2)); // 插入 V2 区块
    ```

4.  **尝试添加 V3 (`ccv3`) 区块**:
    *   为了向前兼容并提供 V3 版本的数据，函数会尝试将输入的 V2 `data` 字符串转换为 V3 格式。
    *   首先，将 V2 JSON 字符串解析为 JavaScript 对象。
    *   然后，向该对象添加 `spec: 'chara_card_v3'` 和 `spec_version: '3.0'` 字段。
    *   将修改后的对象重新序列化为 JSON 字符串，再进行 Base64 编码。
    *   使用 `PNGtext.encode('ccv3', base64EncodedDataV3)` 创建 V3 区块，并同样插入。
    *   这个过程被包裹在一个 `try-catch` 块中，意味着如果 V2 到 V3 的转换或编码失败（例如，输入的 `data` 不是有效的 JSON），将忽略错误，不会影响 V2 区块的写入。
    ```javascript
    try {
        const v3Data = JSON.parse(data); // 解析 V2 JSON
        v3Data.spec = 'chara_card_v3';     // 添加 V3 规范字段
        v3Data.spec_version = '3.0';   // 添加 V3 版本字段

        const base64EncodedDataV3 = Buffer.from(JSON.stringify(v3Data), 'utf8').toString('base64');
        chunks.splice(-1, 0, PNGtext.encode('ccv3', base64EncodedDataV3)); // 插入 V3 区块
    } catch (error) {
        // 忽略在添加 V3 区块时发生的错误
    }
    ```

5.  **重新组装 PNG**: 最后，使用 `./png/encode.js` 模块中的 `encode` 函数，将修改后的区块列表（包含新添加的 `chara` 和可能的 `ccv3` 区块）重新编码成一个新的 PNG 图片缓冲区。
    ```javascript
    // const encode = require('./png/encode.js'); // CommonJS 风格
    const newBuffer = Buffer.from(encode(chunks)); // 将所有区块重新编码为 PNG 图像
    return newBuffer;
    ```

### 2.3 读取角色数据 (`read` 函数详解)

`read` 函数负责从 PNG 图片缓冲区中提取并解码角色数据。

1.  **提取并解码所有 `tEXt` 区块**:
    ```javascript
    const chunks = extract(new Uint8Array(image));
    const textChunks = chunks
        .filter((chunk) => chunk.name === 'tEXt') // 筛选出所有 tEXt 区块
        .map((chunk) => PNGtext.decode(chunk.data)); // 解码每个 tEXt 区块
    ```

2.  **优先查找 `ccv3` 区块**: 系统优先支持 V3 格式。函数会首先查找关键词为 `ccv3` (不区分大小写) 的区块。
    *   如果找到，其文本内容 (Base64 编码的 JSON 字符串) 会被解码 (从 Base64 转回 UTF-8 字符串) 并返回。
    ```javascript
    const ccv3Index = textChunks.findIndex((chunk) => chunk.keyword.toLowerCase() === 'ccv3');
    if (ccv3Index > -1) {
        return Buffer.from(textChunks[ccv3Index].text, 'base64').toString('utf8'); // 解码并返回 V3 数据
    }
    ```

3.  **查找 `chara` 区块**: 如果没有找到 `ccv3` 区块，函数会接着查找关键词为 `chara` (不区分大小写) 的区块。
    *   如果找到，其文本内容同样进行 Base64 解码并返回。
    ```javascript
    const charaIndex = textChunks.findIndex((chunk) => chunk.keyword.toLowerCase() === 'chara');
    if (charaIndex > -1) {
        return Buffer.from(textChunks[charaIndex].text, 'base64').toString('utf8'); // 解码并返回 V2 数据
    }
    ```

4.  **错误处理**:
    *   如果图片中不包含任何 `tEXt` 区块，或者虽然有 `tEXt` 区块但均不包含 `ccv3` 或 `chara` 关键词，函数会打印错误信息到控制台，并抛出一个 "No PNG metadata." 的错误。

### 2.4 文件解析入口 (`parse` 函数)

`parse` 函数是模块对外暴露的主要文件解析接口。

*   **功能**: 它接收一个文件路径 (`cardUrl`) 和可选的格式参数 (`format`)。
*   **格式支持**: 目前，该函数仅支持 `png` 格式。如果 `format` 未定义，则默认为 `png`。任何其他格式都会导致 "Unsupported format" 错误。
*   **读取与解析**: 对于 `png` 文件，它使用 Node.js 的 `fs.readFileSync` 同步读取文件内容到缓冲区，然后调用前面描述的 `read` 函数来提取角色数据。
    ```javascript
    // const fs = require('node:fs'); // CommonJS 风格
    export const parse = async (cardUrl, format) => {
        let fileFormat = format === undefined ? 'png' : format;

        switch (fileFormat) {
            case 'png': {
                const buffer = fs.readFileSync(cardUrl); // 同步读取文件内容
                return read(buffer); // 调用内部 read 函数处理缓冲区
            }
        }
        throw new Error('Unsupported format');
    };
    ```
该模块通过这种方式，实现了角色卡数据在 PNG 图片中的持久化存储和读取，使得角色定义可以和角色图片作为一个整体进行分发和管理。
