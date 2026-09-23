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

# Day 09 · Manifests（下）：data_file 字段与指标

> **阅读材料**：`format/spec.md` 第 708–753 行（Data File Fields）+ 第 754–921 行（Field-level Metrics、Content Stats）
> **预计用时**：45 分钟
> **前置**：Day 08

## 今日目标

- 记住 `data_file` 的必填字段，明确 v2/v3 新增字段的用途。
- 理解 metrics（counts / bounds）的语义与「用于裁剪」的目的。
- 知道 bounds 的三个坑：浮点的 `-0.0` 与 NaN、geography 的跨界包围盒、variant 的路径化 bounds。

## 核心概念

### 1. `data_file` 字段全表

| Field id | 字段 | 说明 | 版本 |
|----------|------|------|------|
| `134` | `content` | `0:DATA`、`1:POSITION DELETES`、`2:EQUALITY DELETES` | v2+ required |
| `100` | `file_path` | 完整 URI（带 FS scheme） | required |
| `101` | `file_format` | `avro` / `orc` / `parquet` / `puffin` | required |
| `102` | `partition` | partition tuple，字段 id 来自该 manifest 的 partition spec | required |
| `103` | `record_count` | 文件行数；对 deletion vector 表示基数（被删行数） | required |
| `104` | `file_size_in_bytes` | 文件总大小 | required |
| `108` | `column_sizes` | 列 id → 该列在磁盘上的字节数（不含 footer；行式格式可留空） | optional |
| `109` | `value_counts` | 列 id → 值个数（含 null 与 NaN） | optional |
| `110` | `null_value_counts` | 列 id → null 个数 | optional |
| `137` | `nan_value_counts` | 列 id → NaN 个数 | optional |
| `125` / `128` | `lower_bounds` / `upper_bounds` | 列 id → binary 序列化的界（Appendix D）；要求 ≤ 所有非 null 非 NaN 值 / ≥ 同样范围 | optional |
| `131` | `key_metadata` | 实现相关的加密 key 元数据 | optional |
| `132` | `split_offsets` | 分裂点偏移（如 Parquet 的 row group 偏移），必须升序 | optional |
| `135` | `equality_ids` | equality delete 用于判等的列 id；`content=2` 时 required，否则必须为 null | v2+ |
| `140` | `sort_order_id` | 该文件的 sort order id（position delete 必须为 null） | optional |
| `142` | `first_row_id` | 文件第一行的 `_row_id`（v3 row lineage） | v3 optional |
| `143` | `referenced_data_file` | 该 entry 的删除全部指向的**唯一** data file；deletion vector **必须**设置 | v2+ optional |
| `144` / `145` | `content_offset` / `content_size_in_bytes` | 指向 Puffin 里某个 blob 的偏移与长度（deletion vector 必须与 Puffin footer 完全一致） | v3 optional |

已废弃、**不要写**的字段：`block_size_in_bytes`（v1 写默认值）、`file_ordinal`、`sort_columns`、`distinct_counts`。

字段 id `141` 在 `data_file` 上是**保留**的。

### 2. metrics 的语义与用途

- 用途：在 plan 阶段**跳过不可能匹配的文件**（data file 与 delete file 用同一套过滤逻辑）。
- 对 delete file：metrics 描述的是**被删除行的值**，要么完整写，要么整体省略（不能只写一部分，否则统计不准）。
- v1–v3 用 `map<column id, value>`；**v4 改用 typed `content_stats` struct**（见下）。
- 两种表示等价：v3 里 map 缺失某 id ≡ v4 里对应字段为 null。

### 3. bounds 的三个坑

**坑一：浮点**

- `-0.0` 必须排在 `+0.0` 之前（IEEE 754 `totalOrder`）。
- **NaN 不能作为 lower/upper bound**。

**坑二：geometry / geography**

- bounds 是包含文件内所有对象的**包围盒**的 X、Y（必需）与 Z、M（可选）坐标。
- 只有 geography 允许 `xmin > xmax`，此时匹配条件是 `x >= xmin OR x <= xmax`（跨 180 度经线的情况）。
- 坐标范围：X ∈ [-180, 180]，Y ∈ [-90, 90]。
- 计算时跳过 null/NaN 维度；若某维度全为 null/NaN 则省略该维度；**X 或 Y 缺失 ⇒ 整个包围盒不产出**。
- v3 用 binary 序列化；v4 用 `geo_lower` / `geo_upper` struct。

**坑三：variant**

- bounds 是 **normalized JSON path → 该路径下界/上界** 的映射，例如 `$`、`$['event_type']`、`$['location']['latitude']`。
- bounds 对该字段所有非 null 值必须成立；数组内元素也要成立。
- **不能为混合 Variant 类型的值写 bounds**（例如既有 int64 又有 string 时必须放弃）。

### 4. v4 的 `content_stats`（了解即可）

- 为「按类型存 stats」设计的容器 struct，嵌在 manifest 里；每个表字段对应一个 stats struct，**按 id 解析**（字段名只是参考）。
- id 分配：stats struct 的 `base-id = 10_000 + 200 * field-id`，结构化字段按 offset 排布（`1: lower_bound`、`2: upper_bound`、`3: tight_bounds`、`4: value_count`、`5: null_value_count`、`6: nan_value_count`、`7: avg_value_size_in_bytes`）。
- 保留的 metadata 字段也有固定区间：`_last_updated_sequence_number` 用 base-id `9000`，`_row_id` 用 `9200`。
- 设计意图：schema evolution / metrics 配置变化时，writer 用「当前的 `content_stats` 类型」读旧 manifest 并做 evolution（如 `int` 读成 `long`），丢弃已删字段的 stats、把新增字段置 null。

## 自测题

1. `record_count` 在 file 与 deletion vector 上分别表示什么？
2. 为什么 delete file 的 metrics 要么全写、要么全不写？
3. `-0.0` 与 `+0.0` 在 bounds 中怎么处理？NaN 可以写进 bounds 吗？
4. geography 的 `xmin > xmax` 表示什么？
5. `content_offset` / `content_size_in_bytes` 主要服务于什么？取值范围不对会怎样？

<details>
<summary>参考答案</summary>

1. file 上是文件行数；deletion vector 上是该向量删除的行数（基数）。
2. 否则基于 metrics 的裁剪会得到错误结果——统计必须能让 reader 安全地跳过文件。
3. `-0.0` 视作小于 `+0.0`；NaN 不允许写入 bounds。
4. 包围盒跨越 ±180 经线，匹配条件为 `x >= xmin OR x <= xmax`。
5. 服务于 deletion vector：在 Puffin 文件中直接定位 blob。它们必须与 Puffin footer 中记录的 `offset` / `length` **完全一致**，否则无法正确读取。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/DataFile.java`、`DeleteFile.java`、`GenericDataFile`（在 `core`）— `data_file` 的内存表示。
- `core/src/main/java/org/apache/iceberg/ManifestReader.java` — metrics 读取与 bounds 解码。
- `core/src/main/java/org/apache/iceberg/Metrics.java`、`MetricsUtil.java` — 写入侧 metrics 计算与截断。
- `api/src/main/java/org/apache/iceberg/expressions/` 下的 `InclusiveMetricsEvaluator` — 用 metrics 做文件裁剪（Day 17）。

## 一句话总结

**`data_file` 用一组 metrics（counts + bounds）让 plan 阶段可以跳过无关文件；bounds 的正确性有严格约束——浮点符号零有顺序、NaN 禁止、geography 允许跨界、variant 按 JSON path 表达。**
