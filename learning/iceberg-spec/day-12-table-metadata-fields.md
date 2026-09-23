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

# Day 12 · Table Metadata（上）：字段与 metadata log

> **阅读材料**：`format/spec.md` 第 1128–1200 行（Table Metadata、Table Metadata Fields）+ 第 1765–1794 行（Appendix C: Table Metadata and Snapshots）
> **预计用时**：40 分钟
> **前置**：Day 03–11

## 今日目标

- 记住 table metadata 的必填字段，特别是 v2 变 required 的那一批。
- 区分 `schema` / `schemas`、`partition-spec` / `partition-specs` 这类「单数 vs 复数」字段。
- 理解 `snapshot-log` 与 `metadata-log` 各自的用途，以及 time travel 该用哪一个。

## 核心概念

### 1. 字段全表（v3 视角）

| 字段 | v1 | v2/v3 | 说明 |
|------|----|-------|------|
| `format-version` | required | required | 高于实现支持版本必须抛异常 |
| `table-uuid` | optional | required | 表身份；refresh 后不匹配必须抛异常 |
| `location` | required | required（v4 起 optional） | 表的基础路径，写文件、写元数据都基于它 |
| `last-sequence-number` | — | required | 已分配的最大 sequence number；读 v1 默认为 0 |
| `last-updated-ms` | required | required | 每次写 metadata 前更新 |
| `last-column-id` | required | required | 分配新 column id 的依据 |
| `schema` | required | 弃用 | 单数形式，被 `schemas` + `current-schema-id` 取代 |
| `schemas` | optional | required | 所有 schema（各带 `schema-id`） |
| `current-schema-id` | optional | required | 当前 schema id |
| `partition-spec` | required | 弃用 | 只有 fields 数组；写数据时用，读数据不用 |
| `partition-specs` | optional | required | 完整 spec 对象列表 |
| `default-spec-id` | optional | required | writer 默认使用的 spec |
| `last-partition-id` | optional | required | 分配新 partition field id 的依据 |
| `sort-orders` | optional | required | 所有 sort order |
| `default-sort-order-id` | optional | required | writer 默认排序 |
| `properties` | optional | optional | string→string 的表属性（如 `commit.retry.num-retries`） |
| `current-snapshot-id` | optional | optional | **必须等于 `refs.main` 指向的 snapshot** |
| `snapshots` | optional | optional | 所有有效 snapshot |
| `snapshot-log` | optional | optional | (timestamp, snapshot-id) 列表，记录 current snapshot 的变更 |
| `metadata-log` | optional | optional | (timestamp, metadata 文件路径) 列表，记录上一个 metadata 文件 |
| `refs` | — | optional | snapshot reference 映射；为 null 时也隐含存在 `main` 分支 |
| `statistics` / `partition-statistics` | optional | optional | 统计文件（Puffin / partition stats file） |
| `next-row-id` | — | v3 required | 大于所有已分配 row id；下一个 snapshot 的 `first-row-id` |
| `encryption-keys` | — | v3 optional | 表加密密钥列表 |

### 2. 两条 log 的区别（time travel 该用哪个）

- `snapshot-log`：**current snapshot 的历史**。每次 `current-snapshot-id` 改变就追加一条 `(last-updated-ms, snapshot-id)`；snapshot 被 expire 时，要删掉比「被过期 snapshot」更早的条目。
- `metadata-log`：**metadata 文件的历史**。每次创建新 metadata 文件时追加一条**上一个** metadata 文件的位置；可通过配置保留固定条数（metadata 文件不会被自动删除，需要显式清理）。

> Appendix F 明确规定：**time travel 必须用 `snapshot-log`**，而不是 snapshots 的 parent-child 链——因为 `current-snapshot-id` 可以被任意设置为某个分支上的 snapshot，两条历史的答案可能不同。`snapshot-log` 缺失时应报出明确的错误信息。

### 3. `next-row-id` 与 `first-row-id`

- `next-row-id` 必须**始终大于**已分配的任何 row id。
- 新 snapshot 提交时推进规则：`next-row-id` 至少要增加「本 snapshot 中会通过继承获得 `first_row_id` 的所有 data file 的 `record_count` 之和」。
- 简单可行的估算法（与 Day 11 的 manifest list 分配一致）：

```
next-row-id = last_assigned.first_row_id + last_assigned.added_rows_count + last_assigned.existing_rows_count
```

### 4. commit 的两个实现方案（都属于 metadata 文件的原子替换）

| 方案 | 流程 | 状态 |
|------|------|------|
| 文件系统（`v<V>.metadata.json`） | 读当前 V → 写唯一临时文件 → rename 到 `v<V+1>.metadata.json`；rename 失败则重试 | **deprecated**，在对象存储/本地文件系统上**不安全**，v4 将移除 |
| metastore（`<V>-<uuid>.metadata.json`） | 写唯一新文件 → 请求 metastore 做 check-and-put 把指针从 V 换到 V+1；失败说明别人已提交 V+1，回到第一步 | 推荐做法（Hive Metastore / REST catalog） |

## 自测题

1. `schema` 与 `schemas` 分别是什么？v2 表还要写 `schema` 吗？
2. 为什么读数据时不需要 `default-spec-id` / `default-sort-order-id`？
3. `table-uuid` 校验失败时应该怎么做？为什么需要这个校验？
4. 做 time travel 到某个时间点，应该查 `snapshot-log` 还是 `metadata-log`？
5. 文件系统方案的 commit 为什么在 S3 上不安全？

<details>
<summary>参考答案</summary>

1. `schema` 是 v1 的单数形式（已弃用，v2 起应省略）；`schemas` 是带 id 的完整列表，配合 `current-schema-id` 使用。
2. 因为读的时候用的是 **manifest 里记录的分区/排序信息**；这两个字段只影响后续写入。
3. 必须抛异常——`table-uuid` 保证「你打开的确实是同一张表」，避免路径复用/误指到别的表。
4. `snapshot-log`（记录 current snapshot 的变化历史）。
5. 因为对象存储不支持可靠的 rename（尤其覆盖语义），「写临时文件 + rename」无法保证原子性与排他性。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/TableMetadata.java`、`TableMetadataParser.java`、`V1Metadata.java` / `V2Metadata.java` / `V3Metadata.java`
- `core/src/main/java/org/apache/iceberg/TableOperations.java` — 提交协议抽象；`HadoopTableOperations` 与 `BaseMetastoreTableOperations`
- `core/src/main/java/org/apache/iceberg/SnapshotLogEntry.java`、`MetadataLogEntry.java`（内嵌在 `TableMetadata` 中）

## 一句话总结

**Table metadata 是整棵树的根：它用带 id 的复数列表（schemas / partition-specs / sort-orders）取代 v1 的单数形式，用 `snapshot-log` 与 `metadata-log` 记录两条不同的历史，并用 `current-snapshot-id` 精确指向 `main` 分支。**
