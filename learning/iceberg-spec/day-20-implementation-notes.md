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

# Day 20 · 实现笔记与路径构造（Appendix F / G）

> **阅读材料**：`format/spec.md` 第 2037–2136 行（Appendix F: Implementation Notes 全文、Appendix G: Geospatial Notes）
> **预计用时**：40 分钟
> **前置**：Day 12（metadata log）、Day 16（row lineage）

## 今日目标

- 掌握路径构造与相对化的推荐做法（v4 相对路径落地的前提）。
- 记住 time travel 必须用 `snapshot-log`，以及 `_change_type` 之外的那些 summary 字段。
- 知道 snapshot id 生成的推荐算法与 `-1` 的历史包袱。

## 核心概念

### 1. Path Construction（写文件时路径怎么拼）

两个表属性控制写入位置：

| 属性 | 默认值 | 含义 |
|------|--------|------|
| `write.metadata.path` | `metadata` | metadata 文件的基础路径 |
| `write.data.path` | `data` | data 文件的基础路径 |

规则（两者一致）：

- 属性值是**绝对路径** ⇒ 直接作为 base；
- 属性值是**相对路径** ⇒ `table location` + `/` + 属性值 拼接为 base。

**Path Relativization（持久化时把绝对路径转相对路径）**：

- 只有当表版本允许时才相对化（v4 起允许）；
- 若文件的绝对路径与 table location 有公共前缀（后面跟 `/`），则存储**相对部分**；
- 否则存储**绝对路径**。

> 拼接规则是「直接用 `/` 连接」，所以规范建议 **table location 不要以路径分隔符结尾**，否则会出现两个 `/`。

### 2. Point in Time Reads（Time Travel）

- Iceberg 有两套历史：`snapshot-log`（current snapshot 的变化）与 snapshots 的 parent-child 链。
- 两者对同一时间戳可能给出**不同的 snapshot id**（例如有人直接把 `current-snapshot-id` 改成某个分支上的 snapshot）。
- **规范要求 time travel 用 `snapshot-log`**；该字段是 optional，缺失或找不到更早的 snapshot 时应报出明确错误。

### 3. Optional Snapshot Summary Fields

summary 里所有值都是**字符串**（如 `"120"`），分两类：

**Metrics（节选）**

| 字段 | 含义 |
|------|------|
| `added-data-files` / `deleted-data-files` / `total-data-files` | 新增 / 删除 / 当前 data file 数 |
| `added-delete-files` / `removed-delete-files`、`added-dvs` / `removed-dvs` | delete file 与 DV 数量 |
| `added-records` / `deleted-records` / `total-records` | 行数变化 |
| `added-position-deletes` / `total-equality-deletes` … | 各类删除记录数 |
| `changed-partition-count` | 有文件增删的分区数 |
| `manifests-created` / `manifests-kept` / `manifests-replaced` / `entries-processed` | manifest 处理情况 |
| `deleted-duplicate-files` | 被删掉的重名文件数 |

**Other fields**

| 字段 | 含义 |
|------|------|
| `wap.id` / `published-wap-id` | Write-Audit-Publish 的 id 与「已发布」的 wap id |
| `source-snapshot-id` | cherry-pick 时的源 snapshot |
| `engine-name` / `engine-version` | 写入引擎与版本 |

### 4. Snapshot ID 的生成

- 应当是**正数**，生成时尽量降低碰撞概率，并在写入前校验不与已有 snapshot 冲突。
- **不建议**只用时间戳生成（碰撞概率高）。
- Java 参考实现：type-4 UUID 的 8 个最高有效字节与 8 个最低有效字节异或，再与 `Long.MAX_VALUE` 相与 ⇒ 伪随机、低碰撞的 long id。
- 「没有 current snapshot」：v1/v2 写 `-1`（等价于省略 / null），**v3 起写 null**；其他实现仍应接受 `-1`。

### 5. 其他实现笔记

- **GZIP 压缩的 metadata**：有的实现要求后缀为 `.gz.metadata.json`，Java 参考实现还能读 `metadata.json.gz`。
- **Position delete file 里的 `row`**：规范允许写，但**目前没有实现真正写入**；Java 实现能读写，但已在 1.11.0 标记 deprecated。

### 6. Appendix G：Geospatial Notes（了解即可）

- `geometry` / `geography` 基于 OGC Simple Feature Access 的 WKT/WKB（支持 XY、XYZ、XYM、XYZM）。
- 坐标顺序固定为 X、Y、Z（可选）、M（可选）：X = 经度/easting，Y = 纬度/northing，Z 通常是高程，M 是第四维（里程、时间戳等，由 CRS 定义）。
- 当前基于 OGC 1.2.1，未来版本只要 WKB 线格式兼容也可以使用。

## 自测题

1. `write.data.path = s3://bucket/base/data` 与 `= data` 时，data file 的写入 base 分别是什么？
2. 什么情况下持久化时会保留绝对路径而不相对化？
3. time travel 应该用 `snapshot-log` 还是 parent-child 链？为什么？
4. Java 参考实现如何生成 snapshot id？为什么不用纯时间戳？
5. `metadata.json.gz` 这个后缀有什么特别之处？

<details>
<summary>参考答案</summary>

1. 前者直接用 `s3://bucket/base/data`；后者是 `table location + "/data"`。
2. 当文件绝对路径与 table location **没有**公共前缀（后跟 `/`）时；或表版本不支持相对路径（v4 之前）。
3. `snapshot-log`。因为 `current-snapshot-id` 可以被直接改到任意 snapshot，两条历史可能给出不同答案，而 time travel 关心的是「那个时刻表指向哪个 snapshot」。
4. type-4 UUID 高低 8 字节异或后与 `Long.MAX_VALUE` 相与，得到低碰撞概率的正 long。
5. 它是 Java 参考实现额外支持的 GZIP metadata 文件后缀（规范推荐的后缀是 `.gz.metadata.json`，两种都能读更兼容）。

</details>

## 与代码对应

- `core/src/main/java/org/apache/iceberg/LocationProviders.java` — `write.data.path` / `write.metadata.path` 的实现
- `core/src/main/java/org/apache/iceberg/TableMetadataParser.java` — metadata log、GZIP 文件读取
- `core/src/main/java/org/apache/iceberg/SnapshotIdGeneratorUtil.java` — snapshot id 生成
- `api/src/main/java/org/apache/iceberg/Snapshot.java`、`core/src/main/java/org/apache/iceberg/SnapshotSummary.java` — summary 字段常量

## 一句话总结

**Appendix F 把「规范没规定但实现必须一致」的细节钉了下来：路径构造与相对化规则、time travel 用 `snapshot-log`、snapshot id 的生成与 `-1` 的历史兼容、以及 summary 字段的完整清单。**
