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

# Day 11 · Manifest Lists

> **阅读材料**：`format/spec.md` 第 993–1046 行（Manifest Lists、manifest_file、field_summary、First Row ID Assignment）
> **预计用时**：35 分钟
> **前置**：Day 08–10

## 今日目标

- 记住 `manifest_file` 的字段（尤其 field id 500–520 这一段）。
- 理解 manifest list 是「plan 的索引」：用它的统计信息跳过整批 manifest。
- 掌握 `first_row_id` 在 manifest list 上的分配规则。

## 核心概念

### 1. manifest list 的角色

- 每个 snapshot 对应**一个** manifest list 文件（Avro），它列出该 snapshot 的所有 manifest。
- 每次 commit 尝试都要**重写** manifest list（因为 manifest 集合总是变化）；写入时把本次的（乐观）sequence number 写进所有新增 manifest 的条目里。
- 用途：包含 added/existing/deleted 文件数、每个分区字段的统计，**用来在 plan 时跳过整个 manifest**。

### 2. `manifest_file` 的字段

| Field id | 字段 | 说明 | v1 / v2+ |
|----------|------|------|----------|
| `500` | `manifest_path` | manifest 文件位置 | required / required |
| `501` | `manifest_length` | 文件字节数 | required / required |
| `502` | `partition_spec_id` | 写该 manifest 的 spec id（必须出现在 table metadata 的 `partition-specs` 中） | required / required |
| `517` | `content` | `0: data`、`1: deletes` | — / required |
| `515` | `sequence_number` | manifest 被加入表时的 sequence number | — / required（读 v1 用 0） |
| `516` | `min_sequence_number` | manifest 中所有存活文件的最小 data sequence number | — / required（读 v1 用 0） |
| `503` | `added_snapshot_id` | 该 manifest 被加入的 snapshot id | required / required |
| `504` `505` `506` | `added_files_count` / `existing_files_count` / `deleted_files_count` | 各状态 entry 数量；为 null 时**假定非零** | optional / required |
| `512` `513` `514` | `added_rows_count` / `existing_rows_count` / `deleted_rows_count` | 各状态 entry 的行数；为 null 时假定非零 | optional / required |
| `507` | `partitions` | `list<field_summary>`，每项对应 manifest 的 partition spec 中一个字段 | optional |
| `519` | `key_metadata` | 加密相关的 key metadata | optional |
| `520` | `first_row_id` | 给该 manifest 中 ADDED data file 分配的起始 `_row_id` | v3 optional |

`field_summary` 的字段：

| Field id | 字段 | 说明 |
|----------|------|------|
| `509` | `contains_null` | 是否至少有一个分区值为 null |
| `518` | `contains_nan` | 是否至少有一个分区值为 NaN |
| `510` / `511` | `lower_bound` / `upper_bound` | 非 null 非 NaN 分区值的界（binary 序列化）；全为 null/NaN 时该值为 null |

> 对比 Day 05 的 bucket / truncate：`field_summary` 让 planner 用**分区条件**就能跳掉整个 manifest，不必打开它。

### 3. `first_row_id` 的分配（v3）

- 已有 manifest 的 `first_row_id` 在写新 manifest list 时必须**原样保留**。
- **delete manifest 的 `first_row_id` 永远是 null**。
- 只给「还没有 `first_row_id` 的 data manifest」分配：第一个这样 manifest 的值 ≥ snapshot 的 `first-row-id`；后续每个的值必须 ≥ 前一个被分配的 manifest 的 `first_row_id` + 该 manifest 中「会通过继承拿到 id 的 data file 行数」。
- 规范给出的简单做法：用 manifest 的 `added_rows_count + existing_rows_count` 来估算：

```
first_row_id = last_assigned.first_row_id + last_assigned.added_rows_count + last_assigned.existing_rows_count
```

- 注意：即使某个 manifest 里**没有 ADDED 文件**，只要它包含「没有 `first_row_id` 的 data file」，也会消耗 id 空间（升级到 v3 后的第一次 commit 就是这么为既有文件补 id 的）。

## 自测题

1. 一个 snapshot 有几个 manifest list？commit retry 时要不要重写？
2. `added_files_count` 为 null 时表示 0 吗？
3. `field_summary` 里的 `lower_bound` 全为 null/NaN 时写什么？
4. delete manifest 的 `first_row_id` 是什么？
5. 为什么需要在 manifest list 上再存一份 `first_row_id`，manifest 里不是有 `first_row_id` 吗？

<details>
<summary>参考答案</summary>

1. 一个。要——每次 commit 尝试都重写，因为 manifest 集合总是变化。
2. 不是，表示「假定非零」（null 表示未知，语义是保守的）。
3. 写 null。
4. 永远是 `null`。
5. 因为 manifest 里的新文件 `first_row_id` 写 null（提交成功前无法确定），需要 manifest list 上的值在**读取时**补全，并且 manifest 的读取顺序决定增量大小；同时这也保证 manifest 可以只写一次就能在 retry 中复用。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/ManifestLists.java`、`ManifestListWriter.java`、`ManifestListReader.java`
- `api/src/main/java/org/apache/iceberg/ManifestFile.java` — planner 看到的 manifest 视图
- `core/src/main/java/org/apache/iceberg/MergingSnapshotProducer.java` — 决定 manifest list 内容（含 `first_row_id` 分配）

## 一句话总结

**manifest list 是 snapshot 的「目录页」：用 counts 与 per-partition 的 field summary 做粗裁剪，用 sequence number 与 `first_row_id` 支撑继承语义，且每次 commit 都整体重写。**
