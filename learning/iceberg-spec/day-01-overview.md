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

# Day 01 · 总览：Iceberg 要解决什么问题

> **阅读材料**：`format/spec.md` 第 68–140 行（Goals、Overview、Optimistic Concurrency、Sequence Numbers、Row-level Deletes、File System Operations、File Locations in Metadata）
> **预计用时**：35 分钟
> **前置**：无

## 今日目标

- 说清楚 Iceberg 和「目录即表」（Hive 风格）的本质差异。
- 背下 metadata tree 的四层结构，并能画出来。
- 理解 6 条设计目标各自对应到规范的哪一部分。
- 知道为什么「乐观并发 + sequence number」是 v2/v3 一切删除语义的基础。

## 核心概念

### 1. 表不是目录，而是一组被显式追踪的文件

Iceberg 追踪的是**单个 data file**，而不是目录：

> This table format tracks individual data files in a table instead of directories.

带来的直接后果：

- 写入是「先写文件，再在 commit 时把文件加入表」，不存在「目录里出现残留文件」这种状态。
- 不需要 listing（对象存储上 listing 很贵），一个 snapshot 包含哪些文件由元数据完全决定。
- 文件一旦写入就不可变，删表/删分区靠删除引用 + 物理清理（`expire snapshots`）。

### 2. metadata tree：四层结构

```
table metadata (JSON, v1.metadata.json)
  └── snapshot (逻辑上内嵌在 table metadata 的 snapshots 列表里)
        └── manifest list (Avro, snap-<snapshot-id>-<attempt>-<uuid>.avro)
              └── manifest (Avro, <uuid>-m<n>.avro)
                    └── data file / delete file (Parquet / Avro / ORC / Puffin)
```

读表只需要 **O(1) 次远程调用**就能开始 plan：table metadata 文件的位置由 catalog 直接给出，snapshot → manifest list → manifest 都是「已知路径的文件」。

### 3. commit 是「原子替换 metadata 指针」

所有表状态变化都会**新建一个 table metadata 文件**，然后原子地把「当前 metadata」指针指向新文件：

- 文件系统实现：写临时文件 + rename（spec 中标注为 **deprecated**、在对象存储上不安全）。
- metastore 实现：catalog 里做 check-and-put（Hive Metastore、REST catalog 等）。

> 这条是后续一切并发正确性的起点，Day 14 会展开。

### 4. Optimistic Concurrency 与隔离级别

- Writer 乐观地认为「我基于的版本在 commit 时仍是最新的」，直接写新 metadata，再尝试替换指针。
- 如果替换失败（有人抢先提交了），**是否可重试取决于本操作的语义**：append 永远可以重放，replace 必须校验被替换的文件是否还在表里，delete 必须校验目标文件是否还在，schema/spec 变更必须校验 schema 没变过。
- 读侧不需要锁：读者用加载 metadata 时的 snapshot，直到 refresh 才会看到新数据 ⇒ serializable isolation。

### 5. Sequence numbers

每个成功 commit 的 snapshot 会被分配一个单调递增的 sequence number：

- snapshot 的 sequence number 会被其创建的 manifest、data file、delete file **继承**；manifest 里显式写 `null`，读时用 manifest list 中的值填充。
- 好处：manifest 写一次就能在 commit retry 中复用，重试时只需重写 manifest list（反正它每次都要重写）。
- v1 表没有这个概念，读 v1 元数据时所有 sequence number 默认为 `0`。

### 6. Row-level deletes（v2 引入）

两类删除，都用 delete file 编码（Day 15–16 展开）：

| 类型 | 语义 | v2 载体 | v3 载体 |
|------|------|---------|---------|
| Position deletes | 按「数据文件路径 + 行位置」删除 | position delete file | deletion vector（Puffin bitmap） |
| Equality deletes | 按「列值相等」删除，如 `id = 5` | equality delete file | 同左 |

删除文件与应用范围由 partition、sequence number 共同约束，规则在 Scan Planning 一节（Day 17）。

### 7. 对文件系统的要求非常低

Iceberg 只要求文件系统支持 **in-place write、seekable reads、deletes**：不要求随机写、不要求 rename（rename 仅用于文件系统表的 commit），所以可以跑在 S3 这类对象存储上。

> 推论：文件写入后不可变；「更新」永远表现为新文件 + 新 metadata。

### 8. Path 的两种形态（v4 引入相对路径）

- **Absolute path**：带 URI scheme（`s3:`、`gs:`、`hdfs:`、`file:`），原样使用。
- **Relative path**：不带 scheme，必须基于 table location 解析；v3 及以前不允许。

相对路径的意义是「表可以整体搬迁而不用重写元数据」，v4 才引入，细节见 Day 20。

## 关键摘录

Goals 六条（原文关键词）：

- **Serializable isolation** — 读用已提交 snapshot，写不会部分可见，读不拿锁。
- **Speed** — plan 是 O(1) 次远程调用，不随分区/文件数增长。
- **Scale** — plan 主要在客户端完成，不依赖中心化 metastore。
- **Evolution** — schema evolution（增删改序改名，含嵌套结构）与 partition spec evolution。
- **Dependable types** — 明确定义的核心类型集合。
- **Storage separation** — 分区是表配置，读时用**数据谓词**（不是分区谓词）来 plan。

## 自测题

1. Iceberg 为什么能做到「读数据文件列表不依赖目录 listing」？
2. `snapshots` 列表存在哪个文件里？manifest 列表存在哪里？为什么后者要单独存文件？
3. 两个 writer 同时基于版本 V 提交，其中一个失败后，什么样的操作可以重试、什么样不可以？
4. 为什么 manifest 里的 sequence number 可以写 `null`？读的时候怎么补全？
5. v1 表里没有 `sequence-number` 字段，读起来应该当作什么值？

<details>
<summary>参考答案（先自己想）</summary>

1. 因为 snapshot 显式列出所有 data/delete file，文件集合由元数据决定，与目录内容无关。
2. `snapshots` 在 table metadata JSON 里；snapshots 对应的 manifest 列表存在单独的 manifest list 文件里（Avro），因为每次 commit 都要重写它，而 snapshot 只是 metadata 里的一个条目。
3. append 无条件可重试；replace/delete 需要校验目标文件仍在表里；schema、partition spec 变更需要校验 schema 未变；基于表达式的 delete 可以直接重放。
4. 因为 snapshot 的 sequence number 在提交成功前才确定；读取时用 manifest list 中的 manifest sequence number 填充 `null`（继承）。
5. 默认为 `0`（Appendix E：Reading v1 metadata for v2 的规定）。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/TableMetadata.java` — table metadata 的内存模型，`TableMetadata.Builder` 是唯一构造入口。
- `core/src/main/java/org/apache/iceberg/TableMetadataParser.java` — JSON 读写（对应 Appendix C）。
- `core/src/main/java/org/apache/iceberg/BaseSnapshot.java` — snapshot 的内存模型。
- `core/src/main/java/org/apache/iceberg/SnapshotProducer.java`、`MergingSnapshotProducer.java` — commit 路径。
- `core/src/main/java/org/apache/iceberg/BaseTable.java`、`BaseTransaction.java` — Engine 侧看到的表/事务实现。
- `core/src/main/java/org/apache/iceberg/TableOperations.java` — 提交抽象：`commit()` 负责原子替换 metadata 指针（`HadoopTableOperations` / `BaseMetastoreTableOperations` 给出两种实现）。

## 一句话总结

**Iceberg 把「表」定义成一棵由不可变文件组成的元数据树，用原子替换 metadata 指针来实现无锁的可串行化提交，用 sequence number 来定义文件之间的新旧关系。**
