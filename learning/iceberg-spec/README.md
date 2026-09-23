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

# Iceberg Spec 21 天学习路线

本目录是一份面向中文读者的 **Apache Iceberg 规范（format spec）** 学习计划：21 天，每天一个文档，每篇 30–45 分钟可以读完。
正文以中文讲解，**代码、字段名、专有名词保留英文**，方便和 `format/spec.md`、Java 实现、引擎配置一一对应。

## 怎么用

1. 每天打开当天的文档（见下方索引，或等 GitHub 的每日提醒 issue）。
2. 对照 `format/spec.md` 的对应行区间阅读**英文原文**；文档里的行号区间就是给你做对照的。
3. 答完文档末尾的**自测题**，答不上来就回读对应小节。
4. 想知道代码里长什么样，看每篇的「与代码对应」小节，按类名在仓库里搜。

> 本地阅读：`format/spec.md`（head of repo）就是规范正文，`.md` 文件之间用相对路径互相链接，所以在 GitHub 网页或本地 Markdown 阅读器里都能直接点开。
> 在线渲染版：<https://iceberg.apache.org/spec/>

## 路线图

| 阶段 | 天数 | 主题 | 你会获得什么 |
|------|------|------|--------------|
| 一、元数据基础 | Day 01 – 07 | Overview、版本演进、Schema、Partitioning、Sort Order、JSON 序列化 | 能够读懂任意一个 Iceberg 表的 metadata JSON 骨架 |
| 二、元数据树 | Day 08 – 14 | Manifests、Snapshots、Manifest Lists、Table Metadata、Commit 协议 | 能够手工追踪「一条数据写进表里，元数据怎么变」 |
| 三、读写路径 | Day 15 – 17 | Delete Formats、Deletion Vectors、Row Lineage、Scan Planning | 能够解释读时删除（read-time merge）和文件裁剪的过程 |
| 四、细节与周边 | Day 18 – 21 | 文件格式要求、哈希、单值序列化、实现笔记、View / Puffin / UDF 等规范 | 知道实现规范时最容易踩坑的边界与周边规范的位置 |

## 每日文档索引

| Day | 主题 | spec 章节 | 文档 |
|-----|------|-----------|------|
| 01 | 总览：Iceberg 要解决什么问题 | Goals / Overview | [day-01-overview.md](day-01-overview.md) |
| 02 | Format Versioning：v1 → v4 的演进 | Format Versioning / Appendix E | [day-02-format-versions.md](day-02-format-versions.md) |
| 03 | Schema 与数据类型 | Schemas / Primitive Types / Nested Types | [day-03-schema-types.md](day-03-schema-types.md) |
| 04 | Schema Evolution 与默认值 | Schema Evolution / Default values / Identifier Fields | [day-04-schema-evolution.md](day-04-schema-evolution.md) |
| 05 | Partitioning 与 Transforms | Partitioning / Partition Transforms | [day-05-partitioning.md](day-05-partitioning.md) |
| 06 | Sort Order | Sorting / Sort Orders | [day-06-sort-orders.md](day-06-sort-orders.md) |
| 07 | 阶段复习：JSON 序列化 | Appendix C | [day-07-json-serialization.md](day-07-json-serialization.md) |
| 08 | Manifests（上）：结构与 manifest_entry | Manifests / Manifest Entry Fields | [day-08-manifests-structure.md](day-08-manifests-structure.md) |
| 09 | Manifests（下）：data_file 字段与指标 | Data File Fields / Field-level Metrics | [day-09-manifests-data-file.md](day-09-manifests-data-file.md) |
| 10 | Snapshots | Snapshots / Snapshot Row IDs | [day-10-snapshots.md](day-10-snapshots.md) |
| 11 | Manifest Lists | Manifest Lists | [day-11-manifest-lists.md](day-11-manifest-lists.md) |
| 12 | Table Metadata（上） | Table Metadata / Table Metadata Fields | [day-12-table-metadata-fields.md](day-12-table-metadata-fields.md) |
| 13 | Table Metadata（下）：分支、标签、保留策略 | Snapshot References / Snapshot Retention Policy | [day-13-branches-and-retention.md](day-13-branches-and-retention.md) |
| 14 | 阶段复习：Commit 协议与序列号 | Optimistic Concurrency / Commit Conflict Resolution / Sequence Numbers | [day-14-commit-protocol.md](day-14-commit-protocol.md) |
| 15 | Delete Formats：position / equality delete | Delete Formats | [day-15-delete-formats.md](day-15-delete-formats.md) |
| 16 | Deletion Vectors 与 Row Lineage（v3） | Deletion Vectors / Row Lineage | [day-16-deletion-vectors-row-lineage.md](day-16-deletion-vectors-row-lineage.md) |
| 17 | Scan Planning：裁剪与去重规则 | Scan Planning | [day-17-scan-planning.md](day-17-scan-planning.md) |
| 18 | 文件格式要求与 32 位哈希 | Appendix A / Appendix B | [day-18-format-requirements.md](day-18-format-requirements.md) |
| 19 | 单值序列化与 Name Mapping | Appendix C / Appendix D | [day-19-serialization.md](day-19-serialization.md) |
| 20 | 实现笔记与路径构造 | Appendix F / Appendix G | [day-20-implementation-notes.md](day-20-implementation-notes.md) |
| 21 | 周边规范速览与后续方向 | view / puffin / udf / expressions / gcm-stream | [day-21-other-specs.md](day-21-other-specs.md) |

## 学习建议

- **先读规范，再看代码**。规范定义的是互操作契约，代码只是其中一种实现；反过来读容易把实现细节误当成规范要求。
- **用真实 metadata 验证**。本地用 Spark 建一张表，`cat` 出 `metadata/v1.metadata.json`、`snap-*.avro`（用 `avro-tools` 或 `parquet-tools` 看内容），和 Day 08–12 里的字段表逐条对照，印象最深。
- **注意版本门槛**。规范里的表格用 v1/v2/v3 列标注字段是 required / optional，这是最容易踩坑的地方：写 v2 表却按 v3 写字段，或读 v1 表却当 required 处理。
- **写不出自测题的答案，就说明那天没读懂**，把那一节原文再读一遍通常就够了。

## 每日提醒是怎么发出的

仓库里的 `.github/workflows/spec-study-daily.yml` 会每天定时读取本目录下的 `day-*.md`，找到**还没提醒过的下一天**，自动创建一个 GitHub issue（标题形如 `[Spec 学习] Day 05 · Partitioning 与 Transforms`）并指派给你，issue 正文就是当天的文档链接，点开即读。

- 手动触发：Actions → `Spec Study Daily Reminder` → Run workflow（可指定分支）。
- 不想用了：直接删掉这个 workflow 文件，issue 只是通知，不影响文档本身。

## 延伸阅读（读完 21 天之后）

- REST Catalog 规范：`site/docs/rest-catalog-spec.md` 与 `open-api/`（客户端与服务端的互操作契约）
- 表加密：`docs/docs/encryption.md` + spec 的 `encryption-keys` / `key-id`
- 分支与标签实践：`docs/docs/branching.md`
- Puffin 文件格式：`format/puffin-spec.md`（统计信息、deletion vector 的宿主文件）
- 视图规范：`format/view-spec.md`（视图是另一棵树：`view-versions` + SQL 表示）
