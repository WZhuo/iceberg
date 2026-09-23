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

# Day 18 · 文件格式要求与 32 位哈希

> **阅读材料**：`format/spec.md` 第 1467–1649 行（Appendix A: Avro / Parquet / ORC、Appendix B: 32-bit Hash Requirements）
> **预计用时**：45 分钟
> **前置**：Day 03、Day 05、Day 09

## 今日目标

- 知道同一份 Iceberg schema 落到 Avro / Parquet / ORC 时的映射规则与**存 field id 的位置**。
- 记住 `bucket[N]` 的哈希规则表与关键测试值。
- 理解为什么「整数提升不能改变哈希」。

## 核心概念

### 1. Avro 映射要点

- **唯一允许的 union**：optional 字段、array 元素、map value 必须包在 `union` 里并带 `null`。
- optional 且无 Iceberg 默认值的字段，Avro 默认值必须为 `null`；有非 null Iceberg 默认值的字段要转换成等价的 Avro 默认值。
- **非 string key 的 map** 必须用「数组 + `map` logical type」表示（元素是 2 字段 record：非 null key + value）；string key 时两种表示都可以。
- 几个 Iceberg 自己的约定：`adjust-to-utc` 属性（缺省 false）、`timestamp-nanos` logical type（Avro 规范里没有）。
- `variant` 用 `record{ metadata, value }` 表示，**这两个字段不分配 field id，按名字访问**。
- `geometry` / `geography` 用 `bytes`（WKB）。

**field id 存哪里**（ID-based column pruning 的前提）：

| 场景 | 存放位置 | 属性名 |
|------|----------|--------|
| struct 字段 | record field 对象 | `field-id` |
| list 元素 | array schema 对象 | `element-id` |
| string map 的 key / value | map schema 对象 | `key-id` / `value-id` |
| 非 string key 的 map | 元素 record 的两个字段 | `field-id`（key 与 value 各一个） |

### 2. Parquet / ORC 映射要点

- **Parquet**：列 id 必须写成 parquet schema 上的 **field ID**；list 必须用 **3-level representation**；`timestamp` 用 `TIMESTAMP_MICROS, adjustToUtc=false`，`timestamptz` 用 `adjustToUtc=true`；decimal 按精度选 `int32`（P ≤ 9）/ `int64`（P ≤ 18）/ `fixed`（用能装下 P 的最小字节数）；`string` 必须是 `UTF8`。
- **ORC**：列 id 存在 ORC type attribute 的 **`iceberg.id`**，是否 required 存在 **`iceberg.required`**（`"true"` / 缺省即 optional）；另有 `iceberg.binary-type`（`UUID`/`FIXED`/`GEOMETRY`/`GEOGRAPHY`）、`iceberg.length`、`iceberg.timestamp-unit`（`MICROS`/`NANOS`）、`iceberg.struct-type=VARIANT` 等约定。
- ORC 的难点是 **schema evolution 是 name-based**，而 Iceberg 是 id-based：做法是 Iceberg 先按自己的规则构造「目标 reader schema」，再改列名让 ORC reader 能映射到 writer schema（规范里给了对照表）。
- ORC 的 `timestamp` 本身是纳秒精度，所以 Iceberg 的 `timestamp`/`timestamptz` writer **必须把纳秒截断到微秒**。

### 3. Appendix B：32 位哈希（`bucket[N]` 的基石）

算法：**32-bit Murmur3, x86 variant, seed = 0**。

| 类型 | 哈希计算 | 示例值 |
|------|----------|--------|
| `int` | `hashLong(long(v))` | `34 → 2017239379` |
| `long` | `hashBytes(littleEndianBytes(v))` | `34L → 2017239379` |
| `decimal(P,S)` | `hashBytes(minBigEndian(unscaled(v)))` | `14.20 → -500754589` |
| `date` | `hashInt(daysFromUnixEpoch(v))` | `2017-11-16 → -653330422` |
| `time` | `hashLong(microsecsFromMidnight(v))` | `22:31:08 → -662762989` |
| `timestamp` / `timestamptz` | `hashLong(microsecsFromUnixEpoch(v))` | `2017-11-16T22:31:08 → -2047944441` |
| `timestamp_ns` / `timestamptz_ns` | 先转成微秒再哈希（保证与微秒类型同值） | 同上 |
| `string` | `hashBytes(utf8Bytes(v))` | `iceberg → 1210000089` |
| `uuid` | `hashBytes(uuidBytes(v))`（大端） | `f79c3e09-677c-4bbd-a479-3f349cb785e7 → 1488055340` |
| `fixed(L)` / `binary` | `hashBytes(v)` | `00 01 02 03 → -188683207` |

三条设计意图（**这是类型提升与分桶兼容的关键**）：

1. **整数与 long 的哈希必须相同**（`int` 按 `hashLong` 计算），这样 `int → long` promotion 不会改变分桶结果。
2. decimal 用「能装下 unscaled value 的最小字节数」的大端补码表示，**与 scale 无关**（scale 是类型的一部分，不是数据值的一部分）。
3. 浮点（当前不用于分桶）若将来需要：`float` 先转 `double` 再哈希，NaN 归一化为 `0x7ff8000000000000L`，`-0.0` 归一化为 `0.0`。
4. `variant` **不定义** 32 位哈希（同一值有多种表示）。

## 自测题

1. Avro 里唯一允许的 union 是什么？
2. Parquet 与 ORC 分别把 Iceberg 的列 id 存在哪里？
3. 为什么 `int` 与 `long` 的哈希结果必须一致？
4. ORC 写 `timestamp` 时为什么要截断纳秒？
5. decimal 的哈希为什么与 scale 无关？

<details>
<summary>参考答案</summary>

1. 「包含 `null` 的 union」——用于 optional 字段、array 元素、map value。
2. Parquet：parquet schema 上的 field ID；ORC：type attribute 里的 `iceberg.id`（外加 `iceberg.required`）。
3. 否则 `int → long` 的类型提升会改变 bucket 分区值，导致旧数据读不出来（分区值不一致）。
4. 因为 Iceberg 的 `timestamp`/`timestamptz` 定义为微秒精度，而 ORC 的 timestamp 是纳秒，必须截断以保持语义一致。
5. 因为哈希输入是 unscaled value 的最小大端字节表示，scale 只影响解释方式，不进入哈希计算。

</details>

## 与代码对应

- `parquet/src/main/java/org/apache/iceberg/parquet/ParquetSchemaUtil.java`、`ParquetTypeVisitor.java`
- `orc/src/main/java/org/apache/iceberg/orc/ORCSchemaUtil.java`（`iceberg.id` 等属性的读写）
- `core/src/main/java/org/apache/iceberg/avro/AvroSchemaUtil.java`
- `api/src/main/java/org/apache/iceberg/types/` 与 `core/src/main/java/org/apache/iceberg/transforms/Bucket.java` — 哈希与分桶实现

## 一句话总结

**Appendix A 规定同一份 Iceberg schema 在三种文件格式中的物理映射与 field id 的存放位置（这是 id-based pruning 的前提），Appendix B 则把 `bucket[N]` 的哈希钉死成 Murmur3 x86 seed 0，并通过「整数同哈希」保证类型提升不改变分桶。**
