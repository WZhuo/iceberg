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

# Day 13 · Table Metadata（下）：分支、标签与保留策略

> **阅读材料**：`format/spec.md` 第 1094–1128 行（Snapshot References、Snapshot Retention Policy）+ Appendix F 中 snapshot id 相关部分
> **预计用时**：35 分钟
> **前置**：Day 12

## 今日目标

- 分清 branch 与 tag 的语义与生命周期。
- 记住 snapshot reference 的五个字段及其对应的表属性。
- 能按规范描述的算法手工推导「哪些 snapshot 会被 expire」。

## 核心概念

### 1. Branch 与 Tag

| 概念 | 语义 | 可变性 |
|------|------|--------|
| **tag** | 给**某一个 snapshot** 起的标签 | 不可变（除非显式重设） |
| **branch** | 具名引用，指向「该分支上的最新 snapshot」 | 可变，提交新 snapshot 时按 commit 冲突解决流程推进引用 |

- 所有引用存放在 table metadata 的 `refs` map 里，key 是引用名（表内唯一）。
- **`main` 分支始终存在**，即使 `refs` 为 null；`refs.main` 必须与 `current-snapshot-id` 一致。
- `main` 分支**永不 expire**。

### 2. Snapshot reference 的字段

| 字段 | 类型 | 适用 | 默认值来源 |
|------|------|------|-----------|
| `snapshot-id` | `long` | 全部 | — （必填） |
| `type` | `string`：`tag` / `branch` | 全部 | — （必填） |
| `min-snapshots-to-keep` | `int`（正数） | 仅 `branch` | 表属性 `history.expire.min-snapshots-to-keep` |
| `max-snapshot-age-ms` | `long`（正数） | 仅 `branch` | 表属性 `history.expire.max-snapshot-age-ms` |
| `max-ref-age-ms` | `long`（正数） | 除 `main` 以外的引用 | 表属性 `history.expire.max-ref-age-ms` |

> 注意粒度：前两个是**分支内「保留多少个 / 多久的 snapshot」**；`max-ref-age-ms` 是**这个引用本身多久没维护就删掉**。

### 3. 保留策略算法（原文照抄，务必能背）

Expire snapshots 时按以下顺序判定：

1. 从「要保留的集合」为空开始。
2. 移除除 `main` 外所有「被引用 snapshot 已超过 `max-ref-age-ms`」的引用。
3. 对每个 branch 和 tag，把它引用的 snapshot 加入保留集合。
4. 对每个 branch，把它的祖先依次加入保留集合，直到满足：该 snapshot **既**超过 `max-snapshot-age-ms`，**又**不在该分支最前面 `min-snapshots-to-keep` 个之内（含分支引用的那个）。
5. 所有不在保留集合里的 snapshot 全部 expire。

配套规则：

- `snapshots` 里列出的都是「有效 snapshot」：其中的 data file 必须都存在于文件系统；**最后一个列出该文件的 snapshot 被 GC 之前，文件不能被删除**。
- 这也是 `expire snapshots` 必须先改 metadata、再删文件的根本原因。

### 4. 典型用法

- **WAP（write-audit-publish）**：写到 staging branch，检查通过后再快进 `main`（Java 侧用 `ManageSnapshots` / `SnapshotManager`）。snapshot summary 里的 `wap.id` / `published-wap-id` 就是为此准备的（Day 20）。
- **cherry-pick**：把某个分支上的 snapshot 应用到另一个分支（summary 里的 `source-snapshot-id` 记录来源）。
- **tag 固定基线**：给「每日快照」「发布版本」打 tag，便于回查。

## 自测题

1. `refs` 为 null 的表还有 `main` 分支吗？`current-snapshot-id` 和 `refs.main` 是什么关系？
2. `min-snapshots-to-keep` 能设置在 tag 上吗？为什么？
3. 一个 branch 上最后 3 个 snapshot 都很老（超过 `max-snapshot-age-ms`），`min-snapshots-to-keep = 2` 时会保留几个？
4. 为什么「最后一个列出某 data file 的 snapshot」被 expire 之前不能删该文件？
5. `max-ref-age-ms` 与 `max-snapshot-age-ms` 分别控制什么？

<details>
<summary>参考答案</summary>

1. 有，`main` 是隐含存在的；`current-snapshot-id` 必须等于 `refs.main` 指向的 snapshot。
2. 不能。它只对 `branch` 有意义（tag 只指向单个 snapshot，不存在「保留几个」的概念）。
3. 保留 2 个（`min-snapshots-to-keep` 优先于年龄条件：必须同时满足「超龄」和「不在前 N 个之内」才会被移除）。
4. 因为「有效 snapshot」的定义包含「其列出的 data file 都还存在」；提前删文件会让该 snapshot 变成无效快照。
5. `max-ref-age-ms` 控制整个引用（分支/标签）保留多久；`max-snapshot-age-ms` 控制分支内单个 snapshot 保留多久。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/SnapshotRef.java`、`SnapshotRefType.java`、`ManageSnapshots.java`、`ExpireSnapshots.java`
- `core/src/main/java/org/apache/iceberg/SnapshotRefParser.java`、`UpdateSnapshotReferencesOperation.java`、`SnapshotManager.java`
- `core/src/main/java/org/apache/iceberg/TableMetadata.java` — `RemoveSnapshots` 中实现保留策略算法
- `docs/docs/branching.md` — 引擎侧使用示例（Spark SQL 的 `CREATE BRANCH` / `CREATE TAG`）

## 一句话总结

**Branch 是可变引用、tag 是快照标签，二者共用一套 `refs` 结构与保留策略；expire 的判定是先按引用收集「必须保留」的集合，再对每个分支沿祖先链按「年龄 + 最少保留数」推进。**
