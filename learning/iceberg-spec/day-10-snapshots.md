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

# Day 10 · Snapshots

> **阅读材料**：`format/spec.md` 第 945–992 行（Snapshots、Snapshot Row IDs）+ 第 2066–2126 行（Appendix F: Optional Snapshot Summary Fields、Assignment of Snapshot IDs）
> **预计用时**：35 分钟
> **前置**：Day 08、Day 09

## 今日目标

- 记住 snapshot 的字段表与 v2/v3 新增字段。
- 分清 `snapshot-id`、`sequence-number`、`parent-snapshot-id`、`current-snapshot-id` 的关系。
- 理解 `first-row-id` / `added-rows` 与 `next-row-id` 的配合。

## 核心概念

### 1. Snapshot 字段表

| v1 | v2 | v3 | 字段 | 说明 |
|----|----|----|------|------|
| required | required | required | `snapshot-id` | 唯一 long id |
| optional | optional | optional | `parent-snapshot-id` | 父 snapshot（无父时省略） |
| — | required | required | `sequence-number` | 单调递增，表示表变更顺序 |
| required | required | required | `timestamp-ms` | 创建时刻，用于 GC 与检查 |
| optional | required | required | `manifest-list` | manifest list 文件位置 |
| optional | — | — | `manifests` | v1 直接列 manifest 路径；有 `manifest-list` 时必须省略 |
| optional | required | required | `summary` | 字符串 map，**必须包含 `operation`** |
| optional | optional | optional | `schema-id` | 创建该 snapshot 时的 current schema id |
| — | — | required | `first-row-id` | 第一条被分配 `_row_id` 的行的起始 id（row lineage） |
| — | — | required | `added-rows` | 本 snapshot 中「被分配 row id 的行数」的上界 |
| — | — | optional | `key-id` | 加密 manifest list key metadata 所用的 key id |

### 2. `operation` 的四种取值（summary 中 required）

| operation | 含义 |
|-----------|------|
| `append` | 只加了文件，没有删文件 |
| `replace` | 加了也删了文件，但**表数据没变**（compaction、格式转换、文件搬迁） |
| `overwrite` | 逻辑覆盖：加了也删了文件，数据变了 |
| `delete` | 删了 data file（内容被逻辑删除）和/或加了 delete file 来删行 |

> 一些操作（如 snapshot expiration）会用它跳过特定 snapshot 的处理。

### 3. 为什么一个 snapshot 会有多个 manifest

1. **fast append**：追加时只写一个新 manifest，而不是改写已有 manifest。
2. **多 partition spec**：每个 manifest 绑定一个 spec，spec evolution 后老 manifest 继续沿用。
3. **并行 planning / 降低改写成本**：大表可以拆到多个 manifest。

### 4. Snapshot Row IDs（v3）

- snapshot 的 `first-row-id` 每次 commit 尝试都被赋值为当时的 `next-row-id`；retry 时要按**当前** `next-row-id` 重新赋值。**即使本次 commit 不分配任何 id 空间，该字段也必须写**。
- `added-rows` 是「被分配 row id 的行数」的**上界**，可以安全地用来推进表的 `next-row-id`；它可能大于本 snapshot 实际新增的行数（会包含部分既有行，例见规范的 Row Lineage Example）。
- 这两个值配合 manifest list 的 `first_row_id`（Day 11）与 manifest entry 的 `first_row_id`（Day 09）构成完整的 row lineage 继承链。

### 5. snapshot id 与 current-snapshot-id 的实现建议（Appendix F）

- snapshot id 应该是**正数**，且写入前要校验不与已有 snapshot 冲突；**不建议**只用时间戳生成（冲突概率高）。
- Java 参考实现：type-4 UUID，把高 8 字节与低 8 字节异或，再与 `Long.MAX_VALUE` 相与，得到低碰撞概率的伪随机 id。
- 「没有 current snapshot」的表示：v1/v2 Java 写 `-1`（等价于省略或 null），**v3 起 Java 不再写 `-1`，统一写 null**；其他实现应能接受 `-1` 当作 null。
- `current-snapshot-id` 必须与 `refs` 中 `main` 分支当前指向的 snapshot 一致。

## 自测题

1. `operation = replace` 与 `overwrite` 的区别是什么？
2. 为什么 append 一个小文件也要新写一个 manifest 而不是改写旧 manifest？
3. v3 中 snapshot 的 `first-row-id` 在 commit retry 时要怎么处理？
4. `added-rows` 为什么可能大于本 snapshot 实际新增的行数？
5. v3 表「没有 current snapshot」应该写什么值？

<details>
<summary>参考答案</summary>

1. `replace` 表示文件被替换但数据内容等价（compaction/搬迁）；`overwrite` 表示数据在逻辑上被覆盖（内容变了）。
2. 因为 manifest 不可变，且 fast append 只需写一个新 manifest，避免读改写已有 manifest。
3. 按当前（最新）的 `next-row-id` 重新赋值，并以新的 manifest list 写出。
4. 因为继承会为「没有 `first_row_id` 的既有 data file」也分配 id，这些既有文件的行数也被计入。
5. `null`（v3 起；v1/v2 的 `-1` 读侧仍应兼容）。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/BaseSnapshot.java`、`api/src/main/java/org/apache/iceberg/Snapshot.java`
- `core/src/main/java/org/apache/iceberg/SnapshotParser.java`、`SnapshotSummary.java`
- `core/src/main/java/org/apache/iceberg/SnapshotIdGeneratorUtil.java` — 参考实现的 snapshot id 生成算法
- `api/src/main/java/org/apache/iceberg/SnapshotUpdate.java`、`SnapshotManager.java` — 更新 snapshot 的操作入口

## 一句话总结

**snapshot 是「某个时刻表的全部文件」的入口：它带上 sequence number、parent 与 summary（含 operation），v3 起还带上 row lineage 的起点（`first-row-id` / `added-rows`）。**
