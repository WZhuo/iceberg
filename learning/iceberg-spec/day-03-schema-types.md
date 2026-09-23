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

# Day 03 · Schema 与数据类型

> **阅读材料**：`format/spec.md` 第 229–294 行（Schemas and Data Types、Nested Types、Semi-structured Types、Primitive Types）+ 第 396–457 行（Column Projection、Identifier Field IDs、Reserved Field IDs）
> **预计用时**：40 分钟
> **前置**：Day 01

## 今日目标

- 记住 primitive types 全集，以及每个类型「added by」哪个版本。
- 理解 **field id 是 schema evolution 的唯一锚点**，名字和顺序都可以变。
- 说清 `struct` / `list` / `map` 对 field id 的要求。
- 知道 metadata columns（`_file`、`_pos` 等）为什么占用 2147483xxx 这段 id。

## 核心概念

### 1. Schema 就是一个 struct type

> A table schema is also a struct type.

字段有：name、id、required/optional、type、可选的 doc、可选的 default value。**id 在整个 table schema 内唯一**（不只是同层唯一）。

### 2. 类型总表（注意 added by version）

| Added by | Type | 备注 |
|----------|------|------|
| v1 | `boolean` `int` `long` `float` `double` | `int → long`、`float → double` 可以 promotion |
| v1 | `decimal(P,S)` | scale 固定，precision ≤ 38 |
| v1 | `date` `time` `timestamp` `timestamptz` | 精度均为 microsecond；`timestamp` 无时区，`timestamptz` 存 UTC |
| v1 | `string` `uuid` `fixed(L)` `binary` | `string` 必须 UTF-8 编码 |
| v3 | `timestamp_ns` `timestamptz_ns` | nanosecond 精度 |
| v3 | `unknown` | 未知类型的占位，必须 optional 且默认 null，不写入数据文件 |
| v3 | `geometry(C)` `geography(C, A)` | 参数 C 是 CRS，A 是 edge-interpolation 算法；默认 `OGC:CRS84` / `spherical` |
| v3 | `variant` | 半结构化，编码遵循 Parquet VariantEncoding（当前 V1） |

> **v3 类型不能出现在 v1/v2 表里**——会破坏 forward-compatibility，这是最典型的版本门槛。

### 3. Nested types 的硬性要求

| 类型 | 结构 | field id 要求 |
|------|------|----------------|
| `struct` | 命名字段元组，字段可 optional/required，可有 doc 和 default | 每个字段一个 id，全表唯一 |
| `list` | element 类型集合；element 可 optional/required | element 有独立 id，全表唯一 |
| `map` | key/value 对 | key 与 value **各有一个 id**；**map key 必须 required**，value 可 optional/required |

细节容易忘的点：

- map 的 key 和 value 都可以是嵌套类型。
- list 的 element 也可以是嵌套类型（`list<struct<...>>` 很常见）。
- 嵌套字段的 id 由 schema 分配，**添加新字段会连带给其所有子字段分配新 id**。

### 4. Column projection：按 id 读，不按名字读

读文件时按 **field id** 选列，所以：

- 重命名列不影响读旧文件（`3: a` 改名为 `3: measurement` 后，仍读同一个物理列）。
- 调换 schema 中列的顺序也不影响读。
- 文件里某个 id 不存在时，按以下优先级补值：
  1. 如果有 `identity` partition transform，从 manifest 的 `partition` 结构里取值（用于 Hive 表 metadata-only migration）；
  2. 用 `schema.name-mapping.default` 属性把「没有 field id 的文件列」按名字映射到 id；
  3. 用 `initial-default`（Day 04）；
  4. 其他情况返回 `null`。

`name-mapping` 的规则要点：`names` 是候选名字列表（可含 Avro alias），`a.b` 是**字面名字**而不是嵌套路径，list/map 分别用 `element` / `key`、`value` 映射。

### 5. Identifier field IDs

- 通过 `identifier-field-ids` 声明「哪些字段唯一标识一行」，用于 upsert / merge 等场景。
- 约束：必须是 primitive；可以嵌在 struct 里，但**不能嵌在 map / list 里**；**不能是 float / double / optional 字段**，也不能嵌套在 optional struct 下（避免 null 参与标识）。
- 注意：Iceberg **不保证**唯一性，这是引擎/数据提供方的责任。
- 这也解释了为什么 equality delete 的 delete columns 有类似限制（Day 15 会看到）。

### 6. Reserved field IDs 与 metadata columns

表字段 id 不得大于 `2147483447`（`Integer.MAX_VALUE - 200`），高段位留给 metadata columns，可以出现在用户 schema 中：

| Field id | Name | 含义 |
|----------|------|------|
| 2147483646 | `_file` | 行所在文件路径 |
| 2147483645 | `_pos` | 行在文件中的位置（从 0 开始） |
| 2147483644 | `_deleted` | 该行是否被删除 |
| 2147483643 | `_spec_id` | 行所属文件使用的 spec id |
| 2147483642 | `_partition` | 行所属的分区 |
| 2147483546 / 2147483545 / 2147483544 | `file_path` / `pos` / `row` | position delete file 的三个字段 |
| 2147483543 … 2147483540 / 2147483539 | `_change_type` / `_change_ordinal` / `_commit_snapshot_id` / `_row_id` / `_last_updated_sequence_number` | changelog 与 row lineage 相关 |

> `_file` + `_pos` 是所有 position delete 语义的基础；`_row_id`、`_last_updated_sequence_number` 是 v3 row lineage 的载体（Day 16）。

## 自测题

1. 表 schema 里两个不同层级的字段可以用同一个 field id 吗？
2. 一个列在数据文件里没有 field id 信息，Iceberg 有哪四种方式给它取值？
3. `map` 的 key 可以为 null 吗？`list` 的 element 呢？
4. 为什么 `double` / `optional` 字段不能作为 identifier field？
5. 为什么用户字段的 id 不能随便用 2147483600？

<details>
<summary>参考答案</summary>

1. 不可以。field id 在**整个 table schema** 内唯一。
2. identity transform 的 partition 值 → `schema.name-mapping.default` 按名字映射 → `initial-default` → null。
3. key 必须 required（不能 null）；element 可以 optional 也可以 required。
4. 因为 identifier 依赖相等性判断，浮点相等与 null 会让「同一行」的判定不可靠/不成立。
5. 高 id 段预留给 metadata columns（`_file`、`_pos`、`_deleted` 等），上界是 `2147483447`。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/types/Types.java` — 类型系统与 `NestedField` 定义。
- `api/src/main/java/org/apache/iceberg/Schema.java` — schema、id 分配、`findField`、name mapping 入口。
- `core/src/main/java/org/apache/iceberg/SchemaParser.java` — Appendix C 的 JSON 序列化。
- `api/src/main/java/org/apache/iceberg/NameMapping.java`、`MappedFields.java` — name mapping 实现。
- `api/src/main/java/org/apache/iceberg/MetadataColumns.java` — `_file`、`_pos`、`_row_id` 等常量与类型。

## 一句话总结

**Schema 是带 id 的类型树：读数据靠 field id（而不是名字和顺序），版本决定了你能用哪些类型，高 id 段留给 metadata columns。**
