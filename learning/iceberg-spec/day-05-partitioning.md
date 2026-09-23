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

# Day 05 · Partitioning 与 Transforms

> **阅读材料**：`format/spec.md` 第 545–635 行（Partitioning、Partition Transforms、Bucket / Truncate 细节、Partition Evolution）
> **预计用时**：45 分钟
> **前置**：Day 03

## 今日目标

- 说清 partition spec 的四个组成部分，以及 partition field id 的特殊地位。
- 全部记住 8 个 transform 的语义、支持的源类型与结果类型。
- 能手算 `bucket[N]` 与 `truncate[W]` 的结果。
- 理解 partition evolution 为什么不能随意改动 field id。

## 核心概念

### 1. Partition spec 的构成

| 组成 | 说明 |
|------|------|
| source column id / source column ids | 取自 table schema；必须是 **primitive**，不能来自 map/list，但可以在 struct 内 |
| partition field id | 在一个 spec 内唯一；**v2 起在表的所有 spec 间唯一**（由 `last-partition-id` 分配） |
| transform | 把源值映射成分区值，见下表 |
| partition name | 分区字段名 |

两个关键约束：

- **同一个 data file 里所有行的 partition values 必须相同**；一个 manifest 只能装使用**同一个 spec** 的文件。
- **两个 spec 等价**的定义是：字段数相同，且对应字段的 source ids、transform、name 都相同。已经存在等价 spec 时，writer **不得**创建新 spec；等价字段必须复用已有 partition field id。

### 2. Partition transform 总表

| Transform | 语义 | 源类型 | 结果类型 |
|-----------|------|--------|----------|
| `identity` | 原值不修改 | 除 `geometry`、`geography`、`variant` 外的任意类型 | 源类型 |
| `bucket[N]` | `murmur3_x86_32(v) & Integer.MAX_VALUE % N` | `int` `long` `decimal` `date` `time` `timestamp` `timestamptz` `timestamp_ns` `timestamptz_ns` `string` `uuid` `fixed` `binary` | `int` |
| `truncate[W]` | 按宽度截断（见下） | `int` `long` `decimal` `string` `binary` | 源类型 |
| `year` | 从 1970 起的年数 | date/timestamp 系 | `int` |
| `month` | 从 1970-01-01 起的月数 | date/timestamp 系 | `int` |
| `day` | 从 1970-01-01 起的天数 | date/timestamp 系 | `date`（读时也接受 `int`） |
| `hour` | 从 1970-01-01 00:00:00 起的小时数 | timestamp 系 | `int` |
| `void` | 永远输出 `null` | 任意 | 源类型或 `int` |

**所有 transform 对 null 输入必须返回 null。**

`void` 的用途：v1 表里「删除」一个分区字段的方式——把 transform 换成 `void`（而不是真删字段）。

### 3. `bucket[N]` 细节

- 哈希算法固定为 **32-bit Murmur3, x86 variant, seed = 0**（逐类型规则见 Appendix B，Day 18）。
- 分桶公式：

```
def bucket_N(x) = (murmur3_x86_32_hash(x) & Integer.MAX_VALUE) % N
```

- 注意是**先丢掉符号位再取模**，保证结果为正。
- `N` 可以通过 spec evolution 改变（表长大后重新分桶）。

### 4. `truncate[W]` 细节

| 类型 | 规则 | 例子 |
|------|------|------|
| `int` / `long` | `v - (v % W)`，余数取正 | `W=10`: `1 → 0`，`-1 → -10` |
| `decimal` | 用 `decimal(W, scale(v))` 计算 | `W=50, s=2`: `10.65 → 10.50` |
| `string` | 取长度为 L 的子串 | `L=3`: `iceberg → ice` |
| `binary` | 取长度为 L 的子数组 | `L=3`: `00 01 02 03 → 00 01 02` |

### 5. Unknown transform

- Writer **不得**用包含未知 transform 的 spec 提交数据。
- Reader：v1/v2 是 should、**v3 起是 must**——忽略该 partition field 用于过滤（但求 partition 相等性时**仍然要用**未知 transform 的结果，见 Day 17）。

### 6. Partition Evolution

- 允许增加、删除、重命名、重排 partition spec 字段；每次产生新的 spec id。
- **不要改变已有 partition field id**：因为 partition field id 会被当作 manifest 里 `partition` struct 的 field id。
- 删除分区字段时的正确做法（v1 兼容规则）：
  1. 不要重排 partition 字段；
  2. 不要真正删除字段，而是把 transform 换成 `void`；
  3. 新增字段只能追加在末尾。
- v1 里 partition field id 未被追踪（参考实现从 1000 开始顺序分配），这导致跨 spec 读 metadata table 时同 id 可能有不同类型——v2 起显式追踪 `last-partition-id` 就是为了解决这个问题。

## 自测题

1. 为什么 manifest 只能包含同一个 partition spec 的文件？
2. `bucket[16]` 对 `34`（int）和 `-1`（int）分别怎么算？为什么公式里要先 `& Integer.MAX_VALUE`？
3. 分区字段 `d = day(ts)` 在 v2 表里，result type 是什么？读时能否接受 `int`？
4. 对 `decimal(9,2)` 的 `10.65` 应用 `truncate[50]` 结果是多少？
5. 为什么删除一个分区字段推荐用 `void` transform 而不是直接删字段？

<details>
<summary>参考答案</summary>

1. 因为 manifest 的 schema（尤其 `partition` struct 的字段 id）是按其 partition spec 生成的，不同 spec 无法共用同一份 schema。
2. 都先用 Murmur3 x86 seed 0 得到 32 位哈希，再 `& Integer.MAX_VALUE`（丢掉符号位保证非负），最后 `% N`。`34`（int 走 `hashLong`）与 `34L` 哈希相同 ⇒ 同桶，这正是 `int → long` promotion 不改变分桶的原因。
3. 结果类型是 `date`；reader 必须同时接受 `int`（按 1970-01-01 起的天数解释）。
4. `10.50`（先按 scale=2 构造 `decimal(50, 2)` 再取模相减）。
5. 因为 partition field id 被 manifest 的 partition struct 复用，直接删字段会破坏历史 manifest 的读法；`void` 保留 id 与位置，只是让新数据落在 `null` 分区。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/PartitionSpec.java`、`UnboundPartitionSpec.java`、`PartitionSpecBuilder.java`
- `api/src/main/java/org/apache/iceberg/transforms/` — 各 transform 实现（`Bucket.java`、`Truncate.java`、`Dates.java`、`Timestamps.java`、`VoidTransform.java`）
- `core/src/main/java/org/apache/iceberg/PartitionSpecParser.java` — Appendix C 的 JSON 格式
- `core/src/main/java/org/apache/iceberg/UpdatePartitionSpec.java` — partition evolution 的校验与 field id 复用逻辑

## 一句话总结

**Partition spec = source ids + transform + 独占的 field id + name；field id 会被 manifest 的 partition struct 复用，所以 evolution 必须保 id、删字段用 `void`、新增只能在末尾。**
