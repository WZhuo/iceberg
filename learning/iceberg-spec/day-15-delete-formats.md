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

# Day 15 · Delete Formats：position / equality delete

> **阅读材料**：`format/spec.md` 第 1350–1466 行（Delete Formats、Position Delete Files、Equality Delete Files、Delete File Stats）
> **预计用时**：40 分钟
> **前置**：Day 09、Day 14

## 今日目标

- 分清三种删除载体（deletion vector / position delete file / equality delete file）与它们的适用版本。
- 记住 position delete file 的字段 id 与排序要求。
- 说清 equality delete 的匹配语义（含 null 的处理）与列限制。

## 核心概念

### 1. 三种载体总览

| 载体 | 语义 | 版本 | 状态 |
|------|------|------|------|
| **Deletion vector (DV)** | 用 bitmap 标记**单个 data file** 内被删除的位置 | v3 | 推荐 |
| **Position delete file** | 用 `(file_path, pos)` 记录被删除的行 | v2 | **v3 已废弃**，v3 表不得新增 |
| **Equality delete file** | 用列值判等删除（如 `id = 3`） | v2 | 一直在用 |

- delete file 本身也是「合法的 Iceberg data file」：必须使用合法格式、schema 与列投影。
- delete file 与 DV 都通过 manifest 追踪，**使用与 data manifest 相同的 schema**，但存放在单独的 delete manifest 里。
- DV 的粒度是「文件位置 + offset + length」，必须记录它所引用的 data file。

### 2. Position Delete Files

存的结构是 `file_position_delete`：

| Field id | 字段 | 类型 | 说明 |
|----------|------|------|------|
| `2147483546` | `file_path` | `string` | 目标 data file 的完整 URI，**必须与该文件在 manifest entry 里的 `file_path` 完全一致** |
| `2147483545` | `pos` | `long` | 行在该 data file 中的序号，**从 0 开始** |
| `2147483544` | `row` | required struct | 被删除行的值（可选列，写的时候一并记录） |

规则细节：

- `row` 列可以**整体省略**；但**只要出现**，就必须是 `required`（所有 delete 条目都要带行值）——这样才能保证统计准确。
- `row` 的 schema 可以是表 schema 的任意子集，但**必须使用与表一致的 field ids**。
- delete 文件中的行**必须按 `file_path` 再按 `pos` 排序**：前者便于列存格式下推过滤，后者便于扫描时增量处理、不必把删除全部装进内存。

### 3. Equality Delete Files

- 存放表列的**任意子集**，使用表的 field ids；真正用于判等的列由 delete file 元数据里的 **`equality_ids`**（Day 09 的 data file 字段 `135`）指定。
- 列的约束与 [identifier fields](day-03-schema-types.md) 相同，**但放宽一条**：允许 optional 列、允许嵌套在 optional struct 下的列（父 struct 为 null 意味着叶子列为 null）。
- 匹配语义：
  - delete file 的一行产生一个等值谓词，多列相当于 `AND`；
  - **null 参与判等**：`delete column` 为 null 时匹配「该行同列为 `null`」，等价于 `col IS NULL`。

举例（原文示例）：

```
表数据：            删除 id = 3 可以写成：
 1: id | 2: category | 3: name      equality_ids=[1]
-------|-------------|---------          1: id
 1     | marsupial   | Koala          -------
 2     | toy         | Teddy            3
 3     | NULL        | Grizzly
 4     | NULL        | Polar

删除 id = 4 AND category IS NULL 需要：equality_ids=[1, 2]
```

- **列被 drop 之后**：仍必须用该列应用历史 equality delete。
- **列是后加的**：读更老的 data file 时按普通投影规则取值（无默认值时是 `null`）。
- 与 row lineage 的交互（Day 16）：用 equality delete 做的更新**不追踪** `_row_id`，视作「删旧行 + 加新行」。

### 4. Delete File Stats

manifest 为 delete file 存与 data file **相同结构**的统计：这些 metrics 描述的是**被删除行的值**。因此规范要求它们**要么完整写、要么整体省略**。

## 自测题

1. v3 表还能新增 position delete file 吗？
2. `file_path` 与目标 data file 的路径必须满足什么关系？
3. position delete file 的行必须如何排序？两个原因分别是什么？
4. `equality_ids = [1, 2]` 是怎么组合成谓词的？其中一列值为 null 会怎样？
5. 一个用于 equality delete 的列后来被 drop 了，读旧数据时还要用它吗？

<details>
<summary>参考答案</summary>

1. 不能。v3 起禁止新增，但已存在的（从 v2 升级来的）position delete file 仍然有效，必须在为该 data file 生成 DV 时合并进去。
2. 必须**完全相等**（同一个完整 URI，含 FS scheme）。
3. 先按 `file_path`，再按 `pos`。前者支持列存格式的文件级过滤下推，后者允许流式扫描、避免把所有删除常驻内存。
4. 两列条件之间是 `AND`；某列为 null 时匹配目标行该列也为 null（等价 `IS NULL`）。
5. 必须继续用它应用删除（否则历史删除会失效）。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/DeleteFile.java`、`DeleteFiles.java`
- `data/src/main/java/org/apache/iceberg/data/DeleteFilter.java`、`GenericDeleteFilter.java`、`BaseDeleteLoader.java` — 读时应用删除
- `core/src/main/java/org/apache/iceberg/deletes/` — DV 的读写实现（如 `BitmapPositionDeleteIndex`、`RoaringPositionBitmap`）
- `api/src/main/java/org/apache/iceberg/MetadataColumns.java` — `file_path` / `pos` / `row` 的常量与类型

## 一句话总结

**v2 用 delete file 表达行级删除：position delete 靠 `(file_path, pos)` 且必须按二者排序，equality delete 靠 `equality_ids` 判等且把 null 当作可匹配值；v3 之后 position delete file 被 deletion vector 取代。**
