## 4. 角色卡数据验证机制 (`src/validator/TavernCardValidator.js`)

为了确保加载的角色卡数据符合预期的格式和规范，从而保证系统的稳定运行和角色行为的一致性，项目采用了一个专门的验证器：`TavernCardValidator`。该验证器位于 `src/validator/TavernCardValidator.js` 文件中。

### 4.1 验证器 `TavernCardValidator` 类

`TavernCardValidator` 是一个 JavaScript 类，其核心职责是根据已知的角色卡规范（目前支持 V1, V2 和 V3 版本）来验证传入的角色卡数据对象。

*   **构造函数**: `constructor(card)`
    *   该构造函数接收一个参数 `card`，这个 `card` 就是待验证的角色卡数据对象。验证器实例会将这个对象存储在 `this.card` 属性中，以供后续的验证方法使用。

### 4.2 多版本验证逻辑 (`validate`, `validateV1`, `validateV2`, `validateV3`)

验证器提供了针对不同版本规范的验证方法。

*   **主验证方法 `validate()`**:
    *   这是外部调用者通常使用的入口方法。
    *   它会依次尝试调用 `this.validateV1()`，`this.validateV2()` 和 `this.validateV3()`。
    *   如果其中任何一个版本的验证成功，`validate()` 方法会返回相应的版本号（数字 `1`、`2` 或 `3`）。
    *   如果所有版本的验证都失败，则返回 `false`。
    *   在开始验证前，它会重置 `this.#lastValidationError` 字段。

*   **V1 版本验证 `validateV1()`**:
    *   此方法用于检查卡片是否符合 V1 规范。V1 卡片数据结构较为扁平，其字段直接存在于卡片对象的顶层。
    *   它主要检查一组 V1 必需字段是否存在于 `this.card` 对象中。
    *   如果所有必需字段都存在，则返回 `true`，否则返回 `false`，并记录第一个缺失的字段到 `#lastValidationError`。
    ```javascript
    // 示例 V1 验证逻辑片段
    validateV1() {
        const requiredFields = ['name', 'description', 'personality', 'scenario', 'first_mes', 'mes_example'];
        return requiredFields.every(field => {
            if (!Object.hasOwn(this.card, field)) { // 检查字段是否直接存在于 this.card
                this.#lastValidationError = field; // 记录验证失败的字段名
                return false;
            }
            return true;
        });
    }
    ```

*   **V2 版本验证 `validateV2()`**:
    *   此方法用于检查卡片是否符合 V2 规范。V2 规范引入了 `spec` 和 `spec_version` 字段，并将核心数据移至 `data` 对象内部。
    *   它通过调用一系列私有辅助方法来完成验证：
        *   `#validateSpecV2()`: 检查 `this.card.spec` 是否严格等于 `'chara_card_v2'`。
        *   `#validateSpecVersionV2()`: 检查 `this.card.spec_version` 是否严格等于 `'2.0'`。
        *   `#validateDataV2()`:
            *   首先检查 `this.card.data` 是否存在且是一个对象。
            *   然后，检查 `this.card.data` 内部是否包含所有 V2 必需的字段，如 `name`, `description`, `personality`, `system_prompt`, `alternate_greetings`, `tags`, `creator`, `character_version`, `extensions` 等。
            *   还会对某些字段的类型进行检查，例如 `alternate_greetings` 和 `tags` 必须是数组，`extensions` 必须是对象。
        *   `#validateCharacterBookV2()`: 如果 `this.card.data.character_book` 存在，则进一步验证其内部结构，例如检查 `extensions` 和 `entries` 字段是否存在，以及它们是否为正确的类型（对象和数组）。
    *   只有当所有这些内部校验都通过时，`validateV2()` 才返回 `true`。

*   **V3 版本验证 `validateV3()`**:
    *   此方法用于检查卡片是否符合 V3 规范。V3 与 V2 类似，也使用 `spec` 和 `spec_version` 字段。
    *   它也通过调用私有辅助方法进行验证：
        *   `#validateSpecV3()`: 检查 `this.card.spec` 是否严格等于 `'chara_card_v3'`。
        *   `#validateSpecVersionV3()`: 检查 `this.card.spec_version` 转换成数字后是否大于等于 `3.0` 且小于 `4.0`。这允许诸如 "3.0", "3.1" 等版本号。
        *   `#validateDataV3()`: 检查 `this.card.data` 是否存在且是一个对象。与 V2 的 `#validateDataV2()` 相比，此处的验证似乎更为宽松，主要关注 `data` 字段的存在性和类型，而没有在这一层级对 `data` 内部的具体字段做强制要求。这可能意味着 V3 的核心数据结构更为灵活，或者详细的内部字段验证由其他模块或更高层逻辑处理。

### 4.3 错误记录 (`lastValidationError`)

为了帮助调试和理解验证失败的原因，验证器内部维护了一个私有字段 `#lastValidationError`。

*   **`#lastValidationError`**: 这是一个私有实例字段，用于存储导致验证失败的具体原因，通常是第一个未能通过校验的字段名（例如 `"name"`, `"data.personality"`, `"spec_version"`）或一个描述性错误信息（如 `"No tavern card data found"`）。
*   **`get lastValidationError()`**: 提供了一个公共的 getter 方法，允许外部代码在 `validate()` 返回 `false` 后，通过访问 `validator.lastValidationError` 来获取具体的错误信息。
*   在每次调用 `validate()` 开始时，`#lastValidationError` 会被重置为 `null`。在各个具体的验证方法（如 `validateV1`, `#validateSpecV2`, `#validateDataV2` 等）内部，如果某个检查失败，相应的方法会将错误原因（通常是字段名）赋给 `#lastValidationError`，然后返回 `false`，从而中断后续的检查。

这种错误记录机制为开发者定位角色卡数据中的问题提供了便利。
