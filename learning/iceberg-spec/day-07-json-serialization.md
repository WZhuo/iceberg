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

# Day 07 · 阶段复习：JSON 序列化（Appendix C）

> **阅读材料**：`format/spec.md` 第 1651–1814 行（Appendix C: JSON serialization 全部小节）
> **预计用时**：40 分钟
> **前置**：Day 03–06

## 今日目标

- 能把 Appendix C 的 5 个小节（Schemas / Partition Specs / Sort Orders / Table Metadata and Snapshots / Name Mapping）串起来，看懂一份真实的 `v1.metadata.json`。
- 记住 JSON 命名的两条硬规则：**key 用 kebab-case**、**optional 字段只在有值时写出**。

## 核心概念

### 1. Schema 的 JSON

Schema 本身就是一个 struct 的 JSON 对象，外加两个字段：

| 字段 | 说明 |
|------|------|
| `schema-id` | v2 起 required |
| `identifier-field-ids` | optional，如 `[1, 2]` |
| `type` / `fields` | 与 `struct` 类型一致 |

field 里的键：`id`、`name`、`required`、`type`、`doc`、`initial-default`、`write-default`。

**类型字符串**（记几个容易写错的）：

| 类型 | JSON |
|------|------|
| `timestamp`（microsecond, 无时区） | `"timestamp"` |
| `timestamptz` | `"timestamptz"` |
| `timestamp_ns` / `timestamptz_ns` | `"timestamp_ns"` / `"timestamptz_ns"` |
| `fixed(16)` | `"fixed[16]"` |
| `decimal(9,2)` | `"decimal(9,2)"`（也接受 `"decimal(9, 2)"`，Postel's Law） |
| `variant` | `"variant"` |
| `geometry(C)` | `"geometry(srid:4326)"` |
| `geography(C, A)` | `"geography(srid:4326, spherical)"` |

nested type 的 JSON 形状：

```json
{ "type": "list", "element-id": 3, "element-required": true, "element": "string" }
{ "type": "map", "key-id": 4, "key": "string", "value-id": 5, "value-required": false, "value": "double" }
```

### 2. Partition Spec 的 JSON

```json
{
  "spec-id": 0,
  "fields": [
    { "source-id": 4, "field-id": 1000, "name": "ts_day", "transform": "day" },
    { "source-id": 1, "field-id": 1001, "name": "id_bucket", "transform": "bucket[16]" }
  ]
}
```

- v3 起多参数 transform 用 `source-ids`（复数），单参数用 `source-id`。
- **manifest 的 key-value metadata 里存的是 `partition-spec`（只有 fields 数组）**，而 table metadata 里存的是完整对象（含 `spec-id`）。这是很常见的混淆点。

### 3. Sort Order 的 JSON

类似 partition spec：`order-id` + `fields`（每个字段含 `source-id`/`source-ids`、`transform`、`direction`、`null-order`）。

### 4. Table Metadata 与 Snapshots

- table metadata JSON 的键就是 Day 12 会详细看的字段：`format-version`、`table-uuid`、`location`、`last-sequence-number`、`last-updated-ms`、`last-column-id`、`schemas`、`current-schema-id`、`partition-specs`、`default-spec-id`、`last-partition-id`、`sort-orders`、`default-sort-order-id`、`properties`、`current-snapshot-id`、`snapshots`、`snapshot-log`、`metadata-log`、`refs`、`statistics`、`partition-statistics` 等。
- snapshot 的 JSON 就是 snapshot 对象本身（Day 10）。
- 命名风格统一：**kebab-case**，例如 `current-snapshot-id`、`snapshot-log`、`default-spec-id`。

### 5. Name Mapping 序列化

对应 Day 03 的 `schema.name-mapping.default`：每个 mapping 对象含 `names`（required，可为空列表）、`field-id`（optional）、`fields`（optional，子字段映射）。

## 动手验证（今天最重要的 15 分钟）

1. 用 Spark 或 `iceberg-core` 写一张小表（可选分区分组），或者用仓库里的 `examples/` 之外任意已有表。
2. 打开 `<table>/metadata/v1.metadata.json`，逐条对照上面的字段列表，圈出哪些字段是 v2 才 required 的。
3. 用 `avro-tools tojson <table>/metadata/snap-*.avro | head` 看 manifest list，再用同样方式看一个 manifest，对照附录 C 与 Day 08/09 的字段表。
4. 试着回答：manifest 的 Avro `meta` 里 `partition-spec` 与 table metadata 里 `partition-specs[0]` 有什么不同？

## 自测题

1. `decimal(9,2)` 的 JSON 表示有几种合法写法？
2. manifest 元数据里保存的 partition spec 是完整对象还是只保存 fields？
3. 为什么 JSON 键要用 kebab-case？optional 字段可以写 `null` 吗？
4. `fixed(16)` 的 JSON 字符串是什么？
5. Name mapping 中 `names: []` 表示什么？

<details>
<summary>参考答案</summary>

1. 两种：`"decimal(9,2)"` 与 `"decimal(9, 2)"`。
2. 只保存 partition fields 数组（外加 manifest 的 `partition-spec-id` 等 key-value metadata）。
3. 因为所有 parser 都以 kebab-case 作为规范形式（`current-snapshot-id` 这类），避免不同实现风格分裂；optional 字段**只在存在时写出**，不写 `null`，读侧把缺失视为未设置。
4. `"fixed[16]"`（注意是方括号，不是圆括号）。
5. 表示该字段只存在于 Iceberg schema，而不在导入的数据文件里（例如 migration 场景）。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/SchemaParser.java`、`PartitionSpecParser.java`、`SortOrderParser.java`、`TableMetadataParser.java`、`SnapshotParser.java`、`NameMappingParser.java`（在 `core/src/main/java/org/apache/iceberg/` 或 `api/` 下按模块分布）
- `core/src/main/java/org/apache/iceberg/SingleValueParser.java` — Appendix D 的单值序列化（Day 19）

## 一句话总结

**Appendix C 定义了元数据在 JSON 里的唯一规范形式：kebab-case 的键、带 id 的类型树、optional 字段按需写出——读一份真实 metadata JSON 是检验前几天所学的唯一标准。**
