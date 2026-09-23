<!--
 - Licensed to the Apache Software Foundation (ASF) under one
 - or more contributor license agreements.  See the NOTICE file
 - distributed with this work for additional information
 - regarding copyright ownership.  The ASF licenses this file
 - to you under the Apache License, Version 2.0 (the
 - "License"); you may not use this file except in compliance
 - with the License.  You may obtain a copy of the License at
 -
 -   http://www.apache.org/licenses/LICENSE-2.0
 -
 - Unless required by applicable law or agreed to in writing,
 - software distributed under the License is distributed on an
 - "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 - KIND, either express or implied.  See the License for the
 - specific language governing permissions and limitations
 - under the License.
 -->

# Day 16 · Deletion Vectors 与 Row Lineage（v3）

> **阅读材料**：`format/spec.md` 第 1369–1384 行（Deletion Vectors）+ 第 458–544 行（Row Lineage、Row lineage assignment / example / for upgraded tables）+ 第 1911–1966 行（Appendix E: Version 3 中 DV 与 row lineage 部分）
> **预计用时**：45 分钟
> **前置**：Day 15

## 今日目标

- 说清 DV 的二进制布局与「每个 data file 最多一个」的维护责任。
- 记住 `_row_id` 与 `_last_updated_sequence_number` 的继承链。
- 知道工程迁移（v2 → v3）时 row id 是怎么补齐的。

## 核心概念

### 1. Deletion Vector 的编码

- DV 使用 Puffin spec 的 **`deletion-vector-v1`** blob 类型存储（Puffin 是「一文件多 blob」的容器，多个 DV 可以放在同一个 Puffin 文件里）。
- 语义：**bit 位置 P 被置位 ⇒ 第 P 行被删除**（`bitmap[i] = 1` 表示该行已删除）。
- 支持正 64 位位置，但为「大多数位置能放进 32 位」的场景做了优化：

```
64 位 position
  ├── 高 32 位 = key      → 选出一个 32 位 Roaring bitmap
  └── 低 32 位 = sub-pos  → 在该 bitmap 中查是否存在
```

- 判断某位置是否被删：先按高 4 字节找 bitmap，再在 bitmap 中查低 4 字节；**找不到对应 key 的 bitmap ⇒ 未被删除**。

### 2. DV 的维护纪律（写侧责任）

- **每个 data file 在一个 snapshot 中最多一个 DV**；写新 DV 时必须把它与已有 DV、以及该 data file 已有的 position delete file **合并**。
- 移除 data file 时，必须同时从 delete manifest 中移除作用于它的 DV（Puffin 文件本身不必重写）。
- 一旦某 data file 有了 DV，**reader 可以安全忽略** 与之匹配的 position delete file。
- delete manifest 用 `file_path` + `content_offset` + `content_size_in_bytes` 定位 blob；这两个值与 Puffin footer 中记录的 `offset` / `length` 必须**完全一致**。
- v3 表**不得新增** position delete file；从 v2 升级来的旧 position delete file 有效，但必须在为该 data file 创建 DV 时被合并进去。

### 3. Row Lineage 的两个字段

| 字段（metadata column） | 含义 |
|------------------------|------|
| `_row_id` | 行的唯一 long id，首次加入表时通过继承分配 |
| `_last_updated_sequence_number` | 最后一次更新该行的 commit 的 sequence number |

- 写入时两者都可以写 `null`（表示「等读时继承」）；文件里**完全没有这两列**时，reader 应视作存在且全为 null。
- 继承链（自顶向下）：

```
table metadata: next-row-id
  → snapshot: first-row-id        （= commit 时的 next-row-id）
  → manifest list 中 manifest: first_row_id
  → manifest entry 中 data file: first_row_id
  → 行: _row_id = data_file.first_row_id + 该行在文件中的 _pos
```

- `_last_updated_sequence_number` 为 null 时，用所在 data file 的 manifest entry 的 **sequence number**（data sequence number）填充。

### 4. 搬移既有行的三条规则（这类场景最容易写错）

1. 行原有的非空 `_row_id` **必须**复制到新 data file；
2. 如果这次写入**修改**了该行，`_last_updated_sequence_number` 要写 `null`（让本次提交的 sequence number 覆盖旧值）；
3. 如果没有修改，原非空 `_last_updated_sequence_number` 要**原样复制**。

> 引擎可以选择把操作建模为「删+插」，也可以建模为「保留 row id 的修改」，但必须遵守上面的规则。

### 5. Equality Delete 不保留 lineage

用 equality delete 做的更新**不追踪** `_row_id`：因为这类引擎在写入前不读旧数据，拿不到原始 row id。规范规定这种更新一律视为「完全删除旧行 + 插入一条新行」。

### 6. 升级到 v3 时的初始化

- 升级时 `next-row-id` 初始化为 `0`，**已有 snapshot 不被修改**（它们的 `first-row-id` 未设置），因此老 snapshot 中 `_row_id` / `_last_updated_sequence_number` 读出来是 null。
- 升级后的**新 snapshot** 必须设置 `first-row-id`，并为 snapshot 中的既有文件与新文件分配 row id：写 manifest list 时为所有 data manifest 分配 `first_row_id`，从而通过继承覆盖到所有 data file。
- 分支之间的注意点：升级后不同分支上的新 snapshot 会基于各自的 `next-row-id` 分配**互不重叠**的 id 区间；同一个 data file 出现在多个分支时，writer 可以复用另一个分支的 `first_row_id`，也可以重新分配（避免大规模重写）。

## 自测题

1. DVs 是如何把一个 64 位位置映射到 bitmap 的？
2. 一个 data file 在一个 snapshot 里能有几个 DV？写第二个时必须做什么？
3. `_row_id` 是怎么算出来的？
4. 把一行搬到新文件且**没有修改**它，`_last_updated_sequence_number` 该写什么？
5. 为什么用 equality delete 做的更新拿不到原有 row id？

<details>
<summary>参考答案</summary>

1. 高 32 位作为 key 选出对应的 32 位 Roaring bitmap，低 32 位作为该 bitmap 内的 sub-position；找不到 key 即视为未删除。
2. 最多一个；必须与已有 DV / position delete file 合并后再写。
3. `data_file.first_row_id + 该行在文件中的位置 (_pos)`；`first_row_id` 逐层从 manifest list → manifest → file 继承。
4. 原样复制原有的非空值（只有「修改了行」才写 null）。
5. 因为这类引擎为避免读放大，在写入前不读取既有数据，因此无法提供原行的 `_row_id`。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/deletes/` — DV 索引与 Roaring bitmap 实现
- `core/src/main/java/org/apache/iceberg/puffin/`（在 `core` / `parquet` 相关模块）— Puffin 文件读写
- `core/src/main/java/org/apache/iceberg/ManifestReader.java` — row lineage 继承（`_row_id` 填充）
- `api/src/main/java/org/apache/iceberg/MetadataColumns.java` — `ROW_ID`、`LAST_UPDATED_SEQUENCE_NUMBER`
- `core/src/main/java/org/apache/iceberg/TableMetadata.java` — `nextRowId` 的管理与递增

## 一句话总结

**DV 用「key + Roaring bitmap」高效表达单文件内的删除位置，且必须由 writer 合并保证「一个 data file 一个 DV」；row lineage 则用「snapshot → manifest list → manifest → file」四级继承把 `_row_id` 与 `_last_updated_sequence_number` 落到行上。**
