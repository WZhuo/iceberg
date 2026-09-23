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

# Day 08 · Manifests（上）：结构与 manifest_entry

> **阅读材料**：`format/spec.md` 第 655–707 行（Manifests、Manifest Entry Fields）
> **预计用时**：35 分钟
> **前置**：Day 05（manifest 与 partition spec 绑定）

## 今日目标

- 知道 manifest 是什么格式、存了哪些 key-value metadata。
- 记住 `manifest_entry` 的 5 个字段与 `status` 的三种取值。
- 理解 `snapshot_id`、`sequence_number`、`file_sequence_number` 的**继承**语义。

## 核心概念

### 1. manifest 本身是什么

- **不可变 Avro 文件**，列出 data file 或 delete file，以及每个文件的 partition tuple、metrics 和追踪信息。
- 一个 manifest **只装 data file 或只装 delete file**（因为 plan 时 delete manifest 会被优先扫描），由 metadata 里的 `content` 区分。
- 一个 manifest **只对应一个 partition spec**；spec 变化后，老文件留在老 manifest，新文件写进新 manifest——这正是「fast append」的实现方式。
- manifest 也是「合法的 Iceberg data file」：必须用合法格式、schema 和列投影规则。

### 2. manifest 的 Avro key-value metadata

| Key | 说明 | 版本要求 |
|-----|------|----------|
| `schema` | 写 manifest 时的 table schema JSON | v1/v2/v3 required |
| `schema-id` | schema id（字符串） | v2+ required |
| `partition-spec` | 分区 spec 的 **fields 数组** JSON | required |
| `partition-spec-id` | spec id（字符串） | v2+ required |
| `format-version` | 表格式版本（字符串） | v2+ required |
| `content` | `"data"` 或 `"deletes"` | v2+ required |

> 读 manifest 时，**必须先读这些 metadata 才能正确解释 `partition` struct**（它的字段 id 来自 partition spec）。

### 3. `manifest_entry` 的字段

| Field id | v1 | v2/v3 | 字段 | 说明 |
|----------|----|-------|------|------|
| `0` | required | required | `status` | `0: EXISTING`、`1: ADDED`、`2: DELETED`；**DELETED 仅作记录，扫描不使用** |
| `1` | required | optional | `snapshot_id` | 文件被加入（或标记删除）时的 snapshot id，为 null 时继承 |
| `3` | — | optional | `sequence_number` | 文件的 **data sequence number**；为 null 且 status = ADDED 时继承 |
| `4` | — | optional | `file_sequence_number` | 文件被**加入表**时的 sequence number；为 null 且 status = ADDED 时继承 |
| `2` | required | required | `data_file` | 文件路径、partition tuple、metrics 等（Day 09） |

**status 的含义**（很关键）：

- `ADDED`：本次 snapshot 加入的文件。
- `EXISTING`：从老 snapshot 继承下来的文件（说明它在更早的 snapshot 中加入）。
- `DELETED`：本次 snapshot 逻辑删除的文件，**仅用于记录**；扫描只读 ADDED / EXISTING。

> `data_file` 被设计成独立 struct，这样 plan 时可以只把 `data_file` 交给任务而不用带 entry 层字段。

### 4. 写入规则

- 文件加入表：entry 记录加入它的 snapshot id，`status = ADDED`。
- 文件被替换/删除：entry 记录删除它的 snapshot id，并**保留原始 data/file sequence number**（不能重算）。
- 「命中 ADDED / EXISTING 的文件路径在同一 snapshot 内出现超过一次 ⇒ 扫描结果未定义」，reader 可以选择报错。

### 5. 继承（inheritance）为什么存在

因为 snapshot 的 sequence number **要等 commit 成功才确定**：

- 新写文件时，manifest 里 sequence number 写 `null`；
- 读取时，用 manifest list 中该 manifest 的 `sequence_number` 填充 `null`；
- 于是 manifest 只需写一次就能在 commit retry 中复用，重试时只需重写 manifest list。

例外：如果一个新文件的**数据逻辑上属于更早的 sequence**（例如把老数据复制进来），必须**显式写入** data sequence number，不能用继承；但 file sequence number 永远由 commit 时赋值。

读 v1 manifest（没有 sequence number 列）时，所有文件的 sequence number 默认 `0`。

## 自测题

1. 一个 manifest 能否同时包含 data file 和 delete file？怎么区分？
2. `status = 2 (DELETED)` 的 entry 会出现在扫描结果里吗？
3. 为什么新写入的 manifest 里可以把 `sequence_number` 写成 `null`？
4. `file_sequence_number` 能否显式指定为更早的值？`sequence_number` 呢？
5. 一个 ADDED 文件在后续 snapshot 中变成 EXISTING 时，sequence number 应该写什么？

<details>
<summary>参考答案</summary>

1. 不能。由 manifest 的 Avro metadata `content` 区分（`"data"` / `"deletes"`）。
2. 不会。DELETED 只是信息性记录，扫描时忽略（其价值在于审计/删除追踪）。
3. 因为 snapshot 的（乐观）sequence number 要等 commit 成功才确定；读时从 manifest list 继承。
4. `file_sequence_number` 不能（它在 commit 成功时被赋值）；`sequence_number` 可以显式指定为一个更早的 data sequence number，用于「数据逻辑上属于更早提交」的场景。
5. 写入原始值（继承或显式指定的值），**不能为 null**——只有 ADDED 才允许 null 继承。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/ManifestFiles.java` — manifest 读写入口（reader/writer 的创建）。
- `core/src/main/java/org/apache/iceberg/ManifestReader.java`、`ManifestWriter.java`、`ManifestEntry.java`（在 `core` 与 `api` 中）
- `api/src/main/java/org/apache/iceberg/ManifestFile.java` — manifest 的抽象（供 plan 使用）
- `core/src/main/java/org/apache/iceberg/MergingSnapshotProducer.java` — 决定哪些 entry 写成 ADDED / EXISTING / DELETED

## 一句话总结

**manifest 是绑定单一份 partition spec 的 Avro 文件，`manifest_entry` 用 status 表达「加入/继承/删除」，用可继承的 sequence number 让 manifest 在 commit retry 中可复用。**
