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

# Day 02 · Format Versioning：v1 → v4 的演进

> **阅读材料**：`format/spec.md` 第 25–67 行（Format Versioning）+ 第 1885–2036 行（Appendix E: Format version changes）
> **预计用时**：35 分钟
> **前置**：Day 01

## 今日目标

- 记住 v1 / v2 / v3 / v4 各引入了什么，特别是**哪些是破坏 forward-compatibility 的**。
- 学会读规范里的「写作要求」表格（required / optional / 空白）以及读侧的兼容规则。
- 理解 spec 里的版本升级策略：**读要宽松，写要严格**。

## 核心概念

### 1. 四个版本各自的主题

| 版本 | 状态 | 主题 | 关键新增 |
|------|------|------|----------|
| v1 | 已采纳 | Analytic data tables | Parquet / Avro / ORC 不可变文件上的大表管理 |
| v2 | 已采纳 | Row-level deletes | delete files（position / equality）、sequence number、更严格的 writer 要求 |
| v3 | 已采纳 | Extended types and capabilities | 新类型（`timestamp_ns`、`unknown`、`variant`、`geometry`、`geography`）、默认值、多参数 transform、row lineage、deletion vectors、表加密 |
| v4 | **开发中，未采纳** | Metadata structure and representation | 相对路径（relative locations） |

> 版本号只在「**新增功能会破坏 forward-compatibility**」，也就是旧 reader 会读错新表时才递增。

### 2. 为什么需要「破坏 forward-compatibility」才升版本

规范明确说：表可以继续按旧版本写，从而保证兼容性——代价是**不能使用新特性**。

两种演进方式对比：

- **v3 的 `write-default`**：只在写时使用，是 forward-compatible 的（旧 writer 会因为没有这个字段而失败，但旧 reader 不受影响）。
- **v3 的 `initial-default`**：旧 reader 读不到这个默认值，如果字段是 required，旧 reader 直接读不了 ⇒ 破坏性。

### 3. 写作要求表怎么读

规范里几乎每个结构都有这样一张表：

| v1 | v2 | v3 | Field |
|----|----|----|-------|
| _required_ | _required_ | _required_ | `snapshot-id` |
| | _required_ | _required_ | `sequence-number` |
| _optional_ | | | `manifests` |

- **空白** = 该版本写的时候**应该省略**。
- **_optional_** = 可写可不写。
- **_required_** = 必须写。

配套的读侧兼容表（Manifest / Manifest list 专用）：

| v1 | v2+ read behavior |
|----|-------------------|
| | 按 optional 读 |
| _optional_ | 忽略该字段 |
| _required_ + v2 _required_ | 缺失就填默认值或抛异常 |

**核心规则**：v1 metadata 允许出现在 v2/v3 表里（升级表不重写元数据树），所以 reader 必须更宽松；但 **metadata JSON 文件可以更严格**——因为 JSON 不会被复用，它一定匹配表版本。例如 v2 表缺 `last-sequence-number` 可以直接抛异常。

### 4. Appendix E 的正确用法

Appendix E 是「升级/降级时该怎么做」的清单，阅读顺序建议：

1. 先看 **Version 2 → Writing v1 metadata**：告诉自己「v1 表里不能写哪些字段」。
2. 再看 **Reading v1 metadata for v2**：哪些字段缺省为 `0`（`last-sequence-number`、`sequence-number`、`min-sequence-number`、manifest entry 的 `sequence_number` / `file_sequence_number`、data file 的 `content`）。
3. 然后看 **Writing v2 metadata**：新增了哪些 required 字段（`table-uuid`、`current-schema-id`、`schemas`、`partition-specs`、`default-spec-id`、`last-partition-id`、`sort-orders`、`default-sort-order-id`）。注意 `schema` / `partition-spec` / `manifests` 这几个 v1 字段**不再写**，改用带 id 的列表形式。
4. v3 / v4 的部分按主题看，等后面几天需要时再回来。

## 容易记错的点

- v1 的 `manifests` 字段（snapshot 里直接列 manifest 路径）在 v2 起被 `manifest-list` 取代，`manifests` **必须省略**。
- `block_size_in_bytes`、`file_ordinal`、`sort_columns`、`distinct_counts` 都是历史字段：v1 写默认值，v2/v3 **不要写**。
- v3 起「读到不认识的 partition transform 必须忽略该分区字段而不报错」是**强制要求**（v1/v2 只是 should）。
- v4 的 `location` 变为 optional（位置可以由 catalog 外部提供）。

## 自测题

1. 判断对错：「表升级到 v2 后，必须重写所有 v1 的 manifest。」为什么？
2. 一个 v2 表的 table metadata JSON 里缺 `table-uuid`，按规范应该怎么处理？缺 `last-sequence-number` 呢？
3. 用 v3 的 `write-default` 会破坏旧 reader 吗？用 `initial-default` 呢？
4. 为什么 manifest list 与 manifest 的字段读取要宽容，而 metadata JSON 可以严格？
5. `manifests` 字段在什么版本可以出现？什么情况下必须省略？

<details>
<summary>参考答案</summary>

1. 错。v1 的 data 与 metadata 文件在升级到 v2 后依然有效，Appendix E 专门规定了 v1 元数据在 v2 中如何补默认值。
2. `table-uuid` 在 v2 是 required，缺失应视为错误；`last-sequence-number` 在 v2 required，但读 v1 元数据时默认 `0`——如果表已经是 v2 且字段缺失，可以抛异常。
3. `write-default` 只影响写入，forward-compatible；`initial-default` 会让旧 reader 读不到既有行的默认值，对 required 字段是破坏性的。
4. 因为 v1 的 manifest / manifest list 会**被 v2+ 表复用**（升级不重写数据），而 metadata JSON 每次 commit 都重写、其内容一定匹配表版本。
5. v1 可以写；v2 起不再写，如果 snapshot 里有 `manifest-list`，`manifests` 必须省略。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/V1Metadata.java`、`V2Metadata.java`、`V3Metadata.java` — 各版本的表级默认值与字段处理差异。
- `core/src/main/java/org/apache/iceberg/TableMetadataParser.java` — v1 元数据的读取兼容逻辑。
- `core/src/main/java/org/apache/iceberg/SnapshotParser.java`、`core/src/main/java/org/apache/iceberg/ManifestFiles.java` — manifest / manifest list 的读写版本分支。

## 一句话总结

**版本号只在破坏 forward-compatibility 时递增；因此读取要向下兼容（v1 元数据允许出现在新版表里），而写入必须严格遵守当前版本的要求表。**
