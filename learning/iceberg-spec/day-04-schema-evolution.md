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

# Day 04 · Schema Evolution 与默认值

> **阅读材料**：`format/spec.md` 第 322–395 行（Default values、Schema Evolution）+ 第 428–436 行（Identifier Field IDs）+ 第 1911–1966 行（Appendix E: Version 3 的 default value 部分）
> **预计用时**：40 分钟
> **前置**：Day 03

## 今日目标

- 分清 `initial-default` 与 `write-default` 的语义差别（这是 SQL default 语义落地的关键）。
- 背下允许的 primitive type promotion，以及「bounds 怎么反推原类型」这张坑表。
- 记住哪些 schema 变更**明确禁止**。

## 核心概念

### 1. 两个默认值

| 默认值 | 作用于 | 何时设置 | 能否修改 |
|--------|--------|----------|----------|
| `initial-default` | **字段被添加之前**写入的所有行（不重写数据文件） | 添加字段时必须设置 | 不能改 |
| `write-default` | 字段添加**之后**写入的行，且 writer 未提供该字段值 | 添加字段时设置（初始等于 `initial-default`） | 可以改（只影响未来写入） |

- 两个默认值都是 **struct field 级别**的属性，顶层 schema 的 struct 字段也可以有。
- 两个都没设时，optional 字段默认 `null`（与旧版本兼容）。
- 新增 **required** 字段：两个默认值都必须非 null；且写数据文件时如果没提供值，要写 `write-default`，如果 required 字段没有 `write-default`，**writer 必须失败**。
- `unknown`、`variant`、`geometry`、`geography` 类型的字段**只能默认 null**。
- 嵌套 struct 的默认值：struct 字段自己的默认值**不包含子字段的默认值**，子字段默认值各记录在子字段上。非 null 的 struct 默认值用**空 struct `{}`** 表示，实际值由各子字段默认值填充：

| `point` 默认 | `x` 默认 | `y` 默认 | 写入的数据 | 读出的结果 |
|--------------|----------|----------|------------|------------|
| `null` | 0 | 0 | （缺字段） | `null` |
| `null` | 0 | 0 | `{"x": 3}` | `{"x": 3, "y": 0}` |
| `{}` | 0 | 0 | （缺字段） | `{"x": 0, "y": 0}` |
| `{}` | 0 | 0 | `{"y": -1}` | `{"x": 0, "y": -1}` |

> 一句话：`initial-default` 决定「老数据的读法」，`write-default` 决定「新数据的写法」。

### 2. Type promotion 白名单

| 原类型 | v1/v2 允许 | v3+ 允许 |
|--------|-----------|----------|
| `unknown` | — | 任意类型 |
| `int` | `long` | `long` |
| `date` | — | `timestamp`、`timestamp_ns` |
| `float` | `double` | `double` |
| `decimal(P,S)` | `decimal(P',S)`，P' > P | 同左 |

额外约束：

- `date → timestamptz` / `timestamptz_ns` **不允许**（promotion 不应改变时区语义）；超出目标类型范围的值必须在运行时失败。
- decimal 只能**加宽 precision**，scale 不变。
- 如果字段是某个 partition field 的 `source-id`，且 promotion 会改变 partition 值，则**禁止** promotion。例如 `bucket[N]` 对 `34` 和 `"34"` 的哈希不同，所以 `int → string` 不行，但 `int → long` 可以（哈希相同）。

### 3. bounds 反推类型（最容易踩的坑）

Avro manifest 里 **bounds 不存类型**，而 promotion 不会重写已有 bounds。所以读 bound 时必须按「当前 schema 类型 + 字节长度」反推原类型：

| 当前类型 | bounds 长度 | 推断原类型 |
|----------|------------|------------|
| `long` | 4 字节 | `int` |
| `long` | 8 字节 | `long` |
| `double` | 4 字节 | `float` |
| `double` | 8 字节 | `double` |
| `timestamp` | 4 字节 | `date` |
| `timestamp` | 8 字节 | `timestamp` |
| `timestamp_ns` | 4 字节 | `date` |
| `timestamp_ns` | 8 字节 | `timestamp_ns` |
| `decimal(P,S)` | 任意 | `decimal(P',S)`，P' ≤ P |

### 4. 结构变更规则

允许：**新增字段、删除字段、重命名字段、调整字段顺序**、按白名单 promotion、struct/list/map 内部任意组合的这些操作。

不允许（会直接报错）：

- 把一组字段**打包成嵌套 struct**，或把嵌套 struct 的字段**提升到父 struct**（`struct<a,b,c> ↔ struct<a, struct<b,c>>`）。
- primitive ↔ struct 互转（含 `map<string,int> ↔ map<string,struct<int>>`）。
- 重命名时**改变 field id**（重命名必须保持 id 不变）。

其他要点：

- 新增字段（含嵌套）分配**新 id**。
- 删除字段只是从当前 schema 移除；**回滚删除只在该字段是 nullable，或当前 snapshot 未变时才可行**。
- 每次 evolution 产生一个**新的 schema id**，加入 `schemas` 列表并成为 `current-schema-id`。
- 结构变更同时必须满足默认值规则：新增字段时 `initial-default` 必须设置且此后不可变，`write-default` 必须设置且可改。

## 自测题

1. 一个表新增 optional 字段 `c`（无默认值），读旧数据时 `c` 是什么？新增 required 字段 `d` 且必须非 null 默认值，直接加行不行？
2. 把 `int` 列改成 `double` 可以吗？改成 `string` 呢？为什么？
3. 为什么读 `long` 类型的 bounds 时要看字节长度？
4. `struct<a, struct<b, c>>` 能否 evolve 成 `struct<a, b, c>`？
5. `initial-default` 被设置后可以修改吗？`write-default` 呢？

<details>
<summary>参考答案</summary>

1. `c` 读为 `null`（两个默认值都未设置时 optional 默认 null）；新增 required 字段时必须同时设置非 null 的 `initial-default` 与 `write-default`，否则非法。
2. `int → double` **不允许**（白名单里没有）；`int → string` 也不允许，且即便存在 general promotion 也不行，因为 `bucket[N]` 的哈希会变。
3. 因为 promotion 不重写历史 bounds，必须按长度推断写文件时的原始类型才能正确解码。
4. 不可以，把嵌套 struct 的字段提升到父 struct 是明确禁止的。
5. `initial-default` 不可修改；`write-default` 可以修改，只影响后续写入。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/SchemaUpdate.java` — evolution 的 API 与合法性校验。
- `api/src/main/java/org/apache/iceberg/types/TypeUtil.java` — type promotion 判断（`isPromotionAllowed`）。
- `core/src/main/java/org/apache/iceberg/MetadataUpdate.java`、`MetadataUpdateParser.java` — `add-schema` / `set-current-schema` 等 REST 可序列化的变更对象。
- `core/src/main/java/org/apache/iceberg/ManifestReader.java` — 读 bounds 时按原类型解码的逻辑。

## 一句话总结

**Schema evolution 只允许「加/删/改名/改序 + 白名单 promotion」，其中新增字段的 `initial-default` 决定老数据怎么读，`write-default` 决定新数据怎么写，而 partition source 字段的 promotion 受哈希稳定性限制。**
