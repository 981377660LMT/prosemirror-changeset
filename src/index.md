好的，我们来对 prosemirror-changeset 这个包进行一次全面、深入的讲解。这个模块是实现“修订追踪”（Track Changes）功能（类似 Word 的修订模式或 Google Docs 的建议模式）的基石。

它的核心任务是：**接收一系列 ProseMirror 的 `Transaction`，并高效地计算、维护一个从某个历史版本到当前版本的、由“删除”和“插入”组成的变更集合。**

我们将从四个核心文件入手，层层递进地分析其设计与实现：

1.  **`change.ts`**: 定义了最基础的数据结构 `Change` 和 `Span`。
2.  **`changeset.ts`**: 核心的 `ChangeSet` 类，负责管理和更新变更集。
3.  **`diff.ts`**: 实现精细化比较的 `computeDiff` 算法。
4.  **`simplify.ts`**: 对变更集进行“美化”，使其更符合人类阅读习惯的 `simplifyChanges`。

---

### 1. change.ts: 变更的原子表示

这个文件定义了两个最基础、最核心的数据结构。

#### `Span<Data>`

可以把 `Span` 理解为一段**带有元数据的连续内容**。

```typescript
export class Span<Data = any> {
  constructor(
    readonly length: number, // 这段内容的长度
    readonly data: Data // 与这段内容关联的元数据
  ) {}
  // ...
}
```

- **`length`**: 这段内容的长度。
- **`data`**: 附加的元数据。在修订追踪场景下，这通常是**作出该变更的用户 ID、时间戳、或者一个唯一的变更 ID**。
- **应用**: 一个插入或删除操作可能由多个不同用户在不同时间完成，`Span` 使得我们可以精确地记录“这段内容是谁在什么时候操作的”。

#### `Change<Data>`

`Change` 类是描述一次“替换”操作的核心。它连接了两个不同版本的文档。

```typescript
export class Change<Data = any> {
  constructor(
    // 在旧文档 A 中的范围
    readonly fromA: number,
    readonly toA: number,
    // 在新文档 B 中的范围
    readonly fromB: number,
    readonly toB: number,
    // 描述被删除内容的 Span 数组
    readonly deleted: readonly Span<Data>[],
    // 描述被插入内容的 Span 数组
    readonly inserted: readonly Span<Data>[]
  ) {}
  // ...
}
```

- **`fromA`, `toA`**: 变更范围在**旧文档（A）**中的起始和结束位置。
- **`fromB`, `toB`**: 变更范围在**新文档（B）**中的起始和结束位置。
- **`deleted`**: 一个 `Span` 数组，详细描述了 `[fromA, toA)` 范围内被删除的内容。所有 `Span` 的 `length` 之和等于 `toA - fromA`。
- **`inserted`**: 一个 `Span` 数组，详细描述了 `[fromB, toB)` 范围内被插入的内容。所有 `Span` 的 `length` 之和等于 `toB - fromB`。

**`Change.merge()`**: 这是一个极其重要的静态方法。它的作用是**合并两个连续的变更集**。假设有 `A -> B` 和 `B -> C` 两个变更集，`merge` 可以将它们合并成一个从 `A -> C` 的单一变更集。它通过一个复杂的并行遍历算法，同步处理两个变更集中的 `Change`，正确地组合 `deleted` 和 `inserted` 的 `Span`，是实现增量更新的基础。

---

### 2. changeset.ts: 变更集的管理者 `ChangeSet`

`ChangeSet` 是整个模块的中心枢纽。它代表了从一个固定的“起始文档”到当前文档的所有变更的集合。

```typescript
export class ChangeSet<Data = any> {
  constructor(
    readonly config: { doc: Node; combine: (dataA: Data, dataB: Data) => Data },
    readonly changes: readonly Change<Data>[]
  ) {}
  // ...
}
```

- `config.doc`: 这个变更集所基于的**起始文档**。
- `config.combine`: 一个函数，用于在合并 `Span` 时决定如何合并它们的 `data`。
- `changes`: 一个 `Change` 对象的有序数组，构成了完整的变更集。

#### 核心方法：`addSteps()`

这是 `ChangeSet` 最核心、最复杂的方法。它的作用是：**在一个现有的 `ChangeSet` 基础上，应用一组新的 `StepMap` (来自一次 `Transaction`)，并生成一个新的 `ChangeSet`。**

其内部工作流程可以分解为以下几个关键步骤：

1.  **步骤转变更 (`StepMap` -> `Change`)**:

    - 遍历传入的 `maps` (`StepMap` 数组)。
    - `StepMap` 自带 `forEach` 方法，可以告诉我们哪些范围被替换了。
    - 将每一个被替换的范围都转换成一个 `Change` 对象。此时的 `Change` 比较粗糙，只知道 A 文档的 `[fromA, toA)` 被 B 文档的 `[fromB, toB)` 替换了。

2.  **合并新变更 (`mergeAll`)**:

    - 上一步可能产生了很多小的、相邻的 `Change` 对象。`mergeAll` 函数使用分治策略，反复调用 `Change.merge`，将这些新的 `Change` 合并成一个规范化的、不重叠的变更列表 `newChanges`。

3.  **合并新旧变更 (`Change.merge`)**:

    - 调用 `Change.merge(this.changes, newChanges, ...)`，将新的变更集 `newChanges` 与 `ChangeSet` 中已有的旧变更集 `this.changes` 合并，得到一个完整的、从最开始的 `startDoc` 到当前 `newDoc` 的变更列表 `changes`。

4.  **最小化变更 (`computeDiff`)**:

    - 这是至关重要的一步优化。合并后的 `changes` 可能存在冗余。例如，用户先输入 "hello"，然后删除了 "ello"，最终的 `Change` 会记录“删除了空，插入了 hello”，然后又记录“删除了 ello，插入了空”。
    - `addSteps` 会遍历所有被新步骤“触碰”到的 `Change` 对象。
    - 对每一个这样的 `Change`，它会调用 `computeDiff(startDoc, newDoc, change)`。
    - `computeDiff` (来自 diff.ts) 会对这个 `Change` 范围内的旧内容和新内容进行精细的 diff 运算。
    - 如果 `computeDiff` 发现旧内容和新内容有相同的部分（例如，在 "he" 和 "hello" 中，"he" 是相同的），它会返回一个更精细的、只包含真正差异部分的 `Change` 列表。
    - 用 `computeDiff` 返回的精细化 `Change` 列表替换掉原来的粗糙 `Change`。

5.  **返回新 `ChangeSet`**:
    - 用经过最小化后的 `changes` 列表和新的 `config` 创建并返回一个新的 `ChangeSet` 实例。整个过程是不可变的。

---

### 3. diff.ts: 精确计算差异

这个文件实现了 `computeDiff` 函数，其核心是**Myers' diff 算法**的一种实现，这是一种非常高效的、寻找两个序列最长公共子序列（LCS）的算法。

#### `tokens()` 函数

- 在进行 diff 之前，必须先将 ProseMirror 的 `Fragment`（文档片段）**“符号化”（tokenize）**。
- `tokens` 函数将一个文档片段转换成一个由数字和字符串组成的数组：
  - **节点开始**: 节点的名称 (如 `"paragraph"`, `"heading"`)。
  - **文本内容**: 每个字符的 `charCodeAt()` 值。
  - **节点结束**: 一个特殊的标记，-1。
- 这种符号化的表示方法使得 diff 算法不仅能比较文本，还能比较文档的结构变化。

#### `computeDiff()` 函数

1.  **预处理**:

    - 接收旧文档片段 `fragA` 和新文档片段 `fragB`。
    - 调用 `tokens` 将它们转换成 `tokA` 和 `tokB`。
    - **快速扫描**: 从两端开始，快速跳过开头和结尾完全相同的部分。这是一个简单但非常有效的优化。

2.  **执行 Myers' Diff**:

    - 在剔除了相同的前后缀后，对中间剩余的部分执行 Myers' diff 算法。
    - 这是一个相当复杂的算法，其基本思想是在一个虚拟的二维网格上寻找一条从 `(0,0)` 到 `(lenA, lenB)` 的路径，这条路径由向右（删除）、向下（插入）和对角线（匹配）移动组成，算法的目标是找到包含最多对角线移动的路径。
    - 为了防止在超大差异上消耗过多性能，算法设置了 `MAX_DIFF_SIZE` 阈值，超过则直接返回整个范围作为一个变更。

3.  **回溯并生成 `Change`**:
    - 当算法找到最优路径后，它会从终点回溯到起点。
    - 回溯路径上的每一个“向右”或“向下”的移动都对应着一个插入或删除，这些被记录下来。
    - 为了避免产生大量零碎的单个字符变更（例如，将 "apple" 替换为 "apply" 可能会产生“删除 e，插入 y”），算法会使用 `minUnchanged` 逻辑，将距离很近的小变更合并成一个大的替换。
    - 最终，将这些变更范围转换成 `Change` 对象数组并返回。

---

### 4. simplify.ts: 让变更更具可读性

`computeDiff` 产生的变更是精确的，但有时不符合人类的直觉。例如，用户在单词 "suggestion" 中间插入了 "s"，变成了 "suggestions"。`diff` 可能会精确地标记出只插入了一个 "s"。但如果用户将 "suggestion" 改成了 "proposal"，`diff` 可能会在两个单词中找到一些零散的共同字母（如 's', 'o'），并把它们标记为未改变，这会让最终的呈现非常混乱。

`simplifyChanges` 的目的就是解决这个问题。

- **核心思想**: 如果一个单词（word）内部同时包含了插入和删除，那么这种零碎的标记是令人困惑的。在这种情况下，宁可将整个单词标记为被删除，然后将新单词标记为被插入。
- **实现**:
  1.  它首先会合并在文档中位置非常接近的 `Change` 对象。
  2.  然后，对于合并后的变更区域，它会检查这个区域是否同时包含插入和删除（但排除了单字符替换这种明确的情况）。
  3.  如果是，它会使用 `isLetter` 函数（一个用于判断字符是否为字母的辅助函数）向变更区域的两端扩展，直到遇到非字母字符（如空格、标点）或文档边界。
  4.  最后，它会创建一个新的、覆盖了整个单词（或连续单词）范围的 `Change` 对象，替换掉原来那个零碎的 `Change`。

### 总结

prosemirror-changeset 的工作流程如同一条精密的流水线：

1.  **输入**: ProseMirror 的 `Transaction` 中的 `StepMap`。
2.  **`ChangeSet.addSteps`**:
    a. 将 `StepMap` 转换成粗略的 `Change` 对象。
    b. 使用 `Change.merge` 将新旧 `Change` 合并。
    c. 对受影响的 `Change` 区域，调用 `computeDiff` 进行“再精炼”。
3.  **`computeDiff`**:
    a. 将文档内容符号化。
    b. 运行 Myers' diff 算法找到最小差异。
    c. 将差异结果转换回更精细的 `Change` 对象。
4.  **`simplifyChanges` (可选的后处理)**:
    a. 遍历最终的 `Change` 列表。
    b. 将不符合人类直觉的、单词内的零碎变更扩展到整个单词。
5.  **输出**: 一个 `ChangeSet` 对象，你可以遍历其中的 `changes` 数组，并根据 `deleted` 和 `inserted` 的 `Span` 信息，在编辑器中将它们渲染成带有高亮或删除线的样式，从而实现“修订追踪”的视觉效果。
