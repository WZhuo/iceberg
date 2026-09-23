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

# Day 14 · 阶段复习：Commit 协议与序列号

> **阅读材料**：`format/spec.md` 第 90–107 行（Optimistic Concurrency、Sequence Numbers、Row-level Deletes）+ 第 1306–1349 行（Commit Conflict Resolution and Retry）+ 第 922–934 行（Sequence Number Inheritance）
> **预计用时**：45 分钟
> **前置**：Day 08–13（这是第一/二阶段的贯通复习）

## 今日目标

- 用一段话讲清「一次写入提交」的完整流程，从写 data file 到替换 metadata 指针。
- 按操作类型说清哪些可以 retry、哪些必须先校验。
- 说清三处 sequence number（snapshot / manifest / data file）的关系，以及 `min_sequence_number` 的用途。

## 核心概念

### 1. 一次 commit 的完整流程

```
1. 写 data file（落盘、不可变、内容里带上所有列，即使与分区值冗余）
2. 写 manifest（entry 的 sequence number 写 null；status=ADDED）
3. 写 manifest list（把本次乐观 sequence number 写到新 manifest 条目上）
4. 构造新的 table metadata（新 snapshot 加入 snapshots、更新 current-snapshot-id / last-sequence-number / next-row-id …）
5. 原子替换 metadata 指针
   ├─ 成功：commit 完成，sequence number 生效
   └─ 失败（别人抢先提交）：判断本操作能否重试，能则基于新版本重做 2–5
```

关键点：**第 2 步写出的 manifest 在 retry 时可以复用**——因为 sequence number 通过继承补全，重试时只需要重写 manifest list（本来每次都要重写）。

### 2. 冲突解决规则（按操作类型）

| 操作类型 | 能否直接重试 | 校验条件 |
|----------|--------------|----------|
| append | ✅ 总是可以 | 无 |
| replace（compaction、格式转换、搬迁） | ⚠️ 有条件 | **被替换的文件必须仍在表里** |
| delete（指定文件） | ⚠️ 有条件 | 目标文件必须仍在表里 |
| delete（基于表达式，如 `where ts < X`） | ✅ 总是可以 | 无 |
| schema 更新 / partition spec 变更 | ⚠️ 有条件 | **base 版本到当前版本之间 schema 未变化** |

> 「写入方自己决定校验什么，从而决定隔离级别（isolation level）」——这是 Iceberg 把隔离级别变成可配置能力的方式。

### 3. Sequence number 的三层

| 位置 | 含义 |
|------|------|
| snapshot 的 `sequence-number` | 本次 commit 的（乐观）序号；成功即生效 |
| manifest list 中每个 manifest 的 `sequence_number` | 该 manifest 被加入表时的序号；也是其内文件继承的来源 |
| manifest list 中每个 manifest 的 `min_sequence_number` | 该 manifest 中**存活文件的最小 data sequence number**（用于判断 manifest 是否可能包含「早于某 delete 文件」的数据） |
| manifest entry 的 `sequence_number` / `file_sequence_number` | 文件的 data / file sequence number |
| data file 的…… | 没有这个字段：它是从 entry 继承来的 |

补充规则：

- **data sequence number** 决定「谁能删谁」：delete file 只能作用于 data sequence number 更小（严格小于时是 equality delete；小于等于时是 position delete）的 data file。
- 新文件默认 `null` → 读时用 manifest 的 seq 填充；把「逻辑上属于更早提交」的数据加入表时必须**显式指定** data sequence number。
- ADDED 之外（EXISTING / DELETED）的 entry，sequence number **必须非 null 且写原值**。
- 读 v1 manifest（无该列）时全部按 `0`。

### 4. 写入方必须遵守的两条纪律

- **所有列都要写进 data file**，即使与 manifest 里的分区值冗余（防止元数据层的 bug 导致数据不可恢复）。
- **不得用包含未知 transform 的 partition spec 提交文件。**

## 自测题

1. 为什么 manifest 可以先写、sequence number 后定？
2. 两个 writer 冲突时，compaction（replace）操作重试前必须校验什么？
3. equality delete 与 position delete 在 sequence number 条件上的差异是什么？
4. `min_sequence_number` 在 plan 时有什么用？
5. 为什么「所有列都写进 data file」是强制要求？

<details>
<summary>参考答案</summary>

1. 因为文件里的 sequence number 可以写 null，读时从 manifest list 继承；这样 manifest 只需写一次，重试只重写 manifest list。
2. 校验**它将删除的文件是否仍在表中**（否则会误删/丢失数据）。
3. equality delete 只作用于 data sequence number **严格小于**自己的 data file；position delete（含 DV）可作用于 **小于等于** 的 data file（允许删掉同一次提交中加入的行）。
4. 用它可以快速判断一个 manifest 是否可能含有被某个 delete file 覆盖的数据——若 manifest 的 `min_sequence_number` 大于 delete 的 sequence number，则整个 manifest 都不需要应用该删除。
5. 因为 manifest 里的分区值等元数据只是「加速与裁剪」的辅助信息，一旦元数据出错，数据文件必须仍能独立恢复出完整数据。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/TableMetadata.java` — `Builder`、`lastAddedSequenceNumber`、`nextRowId`
- `core/src/main/java/org/apache/iceberg/UpdateRequirements.java` — retry 时校验「文件是否还在表里」等条件
- `core/src/main/java/org/apache/iceberg/SnapshotProducer.java`、`MergingSnapshotProducer.java` — commit 主流程与 manifest 复用
- `core/src/main/java/org/apache/iceberg/BaseTransaction.java` — 事务内多次操作共享一次提交
- `core/src/main/java/org/apache/iceberg/ManifestGroup.java` — plan 阶段按 sequence number 组合 data / delete 文件

## 一句话总结

**Iceberg 的提交是「不可变文件 + 原子替换指针」，因此必须用乐观并发 + 按操作类型定义的可重试性来保证正确性，而 sequence number 就是「文件新旧」的唯一判据。**
