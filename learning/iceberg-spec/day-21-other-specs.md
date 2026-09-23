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

# Day 21 · 周边规范速览与后续方向

> **阅读材料**：`format/view-spec.md`、`format/puffin-spec.md`、`format/udf-spec.md`、`format/expressions-spec.md`、`format/gcm-stream-spec.md`、`format/mumbling-spec.md`，以及 `site/docs/rest-catalog-spec.md`
> **预计用时**：45 分钟（今天只求建立索引，不求精读）
> **前置**：Day 01–20

## 今日目标

- 知道每个周边规范解决什么问题、和主 spec 在哪一处咬合。
- 为后续深入挑 1–2 个方向。

## 周边规范一览

| 规范 | 文件 | 解决的问题 | 与主 spec 的接口 |
|------|------|-----------|------------------|
| **View Spec** | `format/view-spec.md` | 视图（不是表）：`view-versions`、SQL 表示、dialect | 与表共享 schema / 序列化风格，用 `view-uuid` + `version-log` 组织历史 |
| **Puffin** | `format/puffin-spec.md` | 一个文件里装多个 blob（statistics、deletion vector） | spec 的 `statistics` / `deletion-vector-v1` / `content_offset` / `content_size_in_bytes` |
| **UDF Spec** | `format/udf-spec.md` | 用户自定义函数的元数据（定义、参数、版本、definition log） | catalog 层面的扩展，与 metadata tree 并列 |
| **Expressions** | `format/expressions-spec.md` | 表达式的结构化表示（value expressions / predicates）与 JSON 序列化 | REST catalog 的 scan planning 请求、谓词序列化 |
| **AES GCM Stream** | `format/gcm-stream-spec.md` | 文件级加密的流式格式扩展（cipher block、AAD、文件长度） | 与主 spec 的 `encryption-keys` / `key-id` 配合 |
| **Mumbling Bitmap** | `format/mumbling-spec.md` | 混淆位图：在 bitmap 上安全地做集合运算（含 PFOR 编码） | 面向 stats/index 类场景 |
| **REST Catalog Spec** | `site/docs/rest-catalog-spec.md` + `open-api/` | 客户端与 catalog 服务端的 HTTP 契约 | 主 spec 里所有 mutation 都要能表达成 REST 的 metadata update |

## 逐个要点

### View Spec

- 视图也有自己的 metadata、`view-versions` 与 `version-log`；每个 version 记录 SQL 表示与 dialect。
- 视图的 schema 同样用 Iceberg 类型系统与 `schema-id` 演进。
- 适合延伸：视图与表在 catalog 侧的统一管理（`ViewCatalog`）。

### Puffin

- 文件结构：`Header` + blobs + `Footer`（footer 里含 footer payload 与 magic）。
- blob 类型是开放的字符串（`apache-datasketches-theta-v1`、`deletion-vector-v1` 等），并带 properties。
- 主 spec 里 DV 的定位就依赖 Puffin footer 里的 `offset` / `length`。

### Expressions

- 分两层：**value expressions**（`reference`、`literal`、函数调用……）与 **predicates**（`and` / `or` / `not` / `is-null` / 比较……）。
- Appendix A 列出了 Iceberg 的函数与 partition transforms 的表达式形式。
- Appendix B 定义了 JSON 序列化——这是 REST catalog 传谓词的基础。
- 适合延伸：`api/src/main/java/org/apache/iceberg/expressions/` 的实现与 `InclusiveMetricsEvaluator` 的结合（Day 17）。

### UDF Spec

- 描述函数的元数据：definition（含参数、返回类型）、definition version、definition log。
- 与表 spec 的关键差异：它是 catalog 侧的命名对象，不参与表的 metadata tree。
- 适合延伸：catalog 如何暴露 UDF 与权限控制。

### AES GCM Stream / Mumbling

- GCM Stream：文件级加密（cipher block、additional authenticated data、文件长度字段），配合主 spec 的加密密钥管理。
- Mumbling：位图格式与 PFOR 编码细节，主要用于 stats / index 场景的高效集合运算。
- 适合延伸：加密表在 Spark/Flink 中的读写路径（`docs/docs/encryption.md`）。

## 后续方向建议

1. **REST Catalog**：从 `open-api/rest-catalog-open-api.yaml` 出发，把「commit table」「load table」两个接口与主 spec 的 metadata update 对上——这是做引擎/服务端最常见的落点。
2. **表加密**：`docs/docs/encryption.md` + spec 的 `encryption-keys` / `key-id` + GCM stream 规范。
3. **Time travel 与分支**：`docs/docs/branching.md` 配合 Day 13 的保留策略，自己动手 expire 几次观察 metadata 变化。
4. **Puffin 与 DV**：读 `parquet` / `core` 中 Puffin 的实现，理解 DV 的读写与合并。
5. **表达式与裁剪**：把 Day 17 的 plan 规则和第 4 条结合，做一次「谓词 → 扫描文件数」的实验。

## 自测题

1. 视图的历史是用什么结构记录的？和表的 `snapshot-log` 有什么区别？
2. deletion vector 为什么要存放在 Puffin 文件里，而不是单独文件？
3. 主 spec 中哪一个表字段直接对应「Puffin blob 的定位信息」？
4. REST catalog 的 scan planning 与 Expressions 规范是什么关系？
5. GCM Stream 与主 spec 的加密字段是怎么配合的？

<details>
<summary>参考答案</summary>

1. 视图用 `version-log`（记录 version 变更）与 `view-versions`；表用 `snapshot-log` 记录 current snapshot 变更，且表的版本由 metadata 文件承载。
2. 因为 Puffin 可以在一个文件里装多个 blob（多个 DV 共享一个容器），并且提供 footer 记录每个 blob 的 offset/length，便于直接定位读取。
3. `content_offset` / `content_size_in_bytes`（与 Puffin footer 中的 `offset` / `length` 必须完全一致）。
4. REST catalog 的 plan 请求要传结构化谓词，其 JSON 形式由 Expressions 规范的 Appendix B 定义。
5. 主 spec 在 table metadata 的 `encryption-keys` 里登记密钥，snapshot 用 `key-id` 指定使用的密钥；GCM Stream 规范定义实际的文件级加密格式。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/view/`（View API）、`core/src/main/java/org/apache/iceberg/view/`
- `core/src/main/java/org/apache/iceberg/puffin/` 或 `parquet` 模块中的 Puffin 读写（按版本不同位置略有差异）
- `api/src/main/java/org/apache/iceberg/expressions/`、`ExpressionsSpec` 相关 parser
- `open-api/rest-catalog-open-api.yaml`、`open-api/rest-catalog-open-api.py`

## 一句话总结

**主 spec 定义「表」的存储契约，其余规范分别扩展「视图、表达式、加密、位图、UDF、catalog 协议」；先把它们和主 spec 的咬合点记住，再挑一个方向精读即可。**
