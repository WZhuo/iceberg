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

# Day 19 · 单值序列化与 Name Mapping（Appendix C / D）

> **阅读材料**：`format/spec.md` 第 1815–1884 行（Appendix D: Binary / Bound / JSON single-value serialization）+ 第 1795–1814 行（Appendix C: Name Mapping Serialization）
> **预计用时**：35 分钟
> **前置**：Day 07、Day 09

## 今日目标

- 分清三种单值序列化：**binary**（manifest metrics / bounds）、**JSON**（metadata 中的默认值等）、**bound serialization**（对 binary 的补充规则）。
- 记住 JSON 单值序列化里几个「反直觉」的约定。
- 会手写/读懂 name mapping 的 JSON。

## 核心概念

### 1. Binary single-value serialization（用于 `lower_bounds` / `upper_bounds` 等）

| 类型 | 二进制表示 |
|------|------------|
| `boolean` | `0x00` = false，非零 = true |
| `int` / `date` | 4 字节 little-endian |
| `long` / `time` / `timestamp` / `timestamptz` / `timestamp_ns` / `timestamptz_ns` | 8 字节 little-endian |
| `float` / `double` | 4 / 8 字节 little-endian |
| `string` | UTF-8 字节（**不带长度**） |
| `uuid` | 16 字节 **big-endian** |
| `fixed(L)` / `binary` | 原始字节（不带长度） |
| `decimal(P,S)` | unscaled value 的**最小长度**大端补码 |
| `geometry` / `geography` | WKB |
| `struct` / `list` / `map` / `variant` / `unknown` | **不支持**（bound 另见下节） |

时间语义提醒：

- `date`：1970-01-01 起的天数。
- `time`：午夜起的微秒数。
- `timestamp`：1970-01-01 00:00:00.000000 起的微秒（无时区）。
- `timestamptz`：1970-01-01 00:00:00.000000 **UTC** 起的微秒。

### 2. Bound serialization（对上面规则的例外）

`lower_bounds` / `upper_bounds` 用的就是 binary single-value 序列化，除了：

| 类型 | 特殊规则 |
|------|----------|
| `geometry` / `geography` | 一个点：`x:y:z:m` 四段 8 字节 little-endian IEEE 754 坐标拼接；x、y 必需；z、m 都无 ⇒ `x:y`；只缺 m ⇒ `x:y:z`；**只缺 z ⇒ `x:y:NaN:m`** |
| `variant` | v1 metadata 的 Variant 编码 + 一个 Variant object；object 的 key 是归一化 JSON path（如 `$['location']['latitude']`），value 是该字段的上下界 |

### 3. JSON single-value serialization（metadata 里的默认值等）

| 类型 | JSON | 注意 |
|------|------|------|
| `decimal(P,S)` | JSON **string**：`"14.20"`、`"2E+20"` | 正 scale 用小数点位数表达；**负 scale 用科学计数法且指数等于 -scale** |
| `date` | `"2017-11-16"` | ISO-8601 |
| `time` | `"22:31:08.123456"` | ISO-8601，微秒精度 |
| `timestamp` | `"2017-11-16T22:31:08.123456"` | **不得**带时区偏移 |
| `timestamptz` | `"2017-11-16T22:31:08.123456+00:00"` | **必须**带偏移且必须是 `+00:00` |
| `timestamp_ns` / `timestamptz_ns` | 同上的纳秒版本 | 无时区 / 必须 `+00:00` |
| `uuid` | **小写**字符串 | `"f79c3e09-…"` |
| `fixed(L)` / `binary` | **十六进制字符串** | `"000102ff"` |
| `struct` | JSON object，**key 是 field id** | `{"1": 1, "2": "bar"}` |
| `list` | JSON array | `[1, 2, 3]` |
| `map` | `{ "keys": [...], "values": [...] }` | 不是普通 JSON object |
| `geometry` / `geography` | WKT 字符串 | `"POINT (30 10)"` |

> 两个常见的实现 bug：`timestamptz` 的 `+00:00` 必须显式写出、且只能写 `+00:00`；`struct` 的 JSON key 是 **field id 而不是字段名**。

### 4. Name Mapping 序列化

`schema.name-mapping.default` 的值是 field mapping 对象列表：

| 字段 | JSON |
|------|------|
| `names` | 字符串列表，如 `["latitude", "lat"]` |
| `field-id` | int，如 `1` |
| `fields` | 子字段映射列表 |

```json
[ { "field-id": 1, "names": ["id", "record_id"] },
  { "field-id": 2, "names": ["data"] },
  { "field-id": 3, "names": ["location"], "fields": [
      { "field-id": 4, "names": ["latitude", "lat"] },
      { "field-id": 5, "names": ["longitude", "long"] }
    ] } ]
```

这与 Day 03 的规则一一对应：`a.b` 是**字面名字**；list 用 `element`，map 用 `key` / `value` 作为子映射名；不在 Iceberg schema 里的字段可以省略 `field-id`。

## 自测题

1. `long` 类型 bounds 的字节序是什么？`uuid` 呢？
2. `geometry` 的 bound 序列化中，`x:y:NaN:m` 表示什么？
3. `timestamptz` 的 JSON 单值必须写成什么样？
4. `map<string, int>` 的 JSON 单值形式是什么？
5. name mapping 里 `names` 有两个名字 `["latitude", "lat"]` 表示什么？

<details>
<summary>参考答案</summary>

1. `long` 是 8 字节 little-endian；`uuid` 是 16 字节 big-endian。
2. 表示 Z 维度未设置（缺失），而 M 维度有值——所以用 NaN 占住 Z 的位置。
3. 必须带时区偏移，且偏移必须是 `+00:00`，例如 `"2017-11-16T22:31:08.123456+00:00"`。
4. `{ "keys": ["a", "b"], "values": [1, 2] }`。
5. 该字段在不同数据文件里可能叫不同名字（例如 Avro aliases），两个名字都映射到同一个 field id。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/SingleValueParser.java` — JSON 单值序列化
- `core/src/main/java/org/apache/iceberg/ManifestReader.java`、`Metrics.java` — binary bounds 的读写
- `core/src/main/java/org/apache/iceberg/mapping/NameMappingParser.java`、`api/src/main/java/org/apache/iceberg/mapping/NameMapping.java`

## 一句话总结

**指标用的 bounds 走 little-endian 的 binary 序列化（并有三类特殊规则），metadata 里的默认值走 JSON 单值序列化（时区、十六进制、field id 作 key 都是硬约束），name mapping 则把「无 id 的旧文件列」按名字映射回 field id。**
