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

# Day 06 · Sort Order

> **阅读材料**：`format/spec.md` 第 636–654 行（Sorting）+ 第 1733–1764 行（Appendix C: Sort Orders）
> **预计用时**：25 分钟
> **前置**：Day 05（transforms 是共用的）

## 今日目标

- 说清 sort order 的四个组成部分，以及 order id `0` 的特殊含义。
- 知道 sort order 是怎么和 data file 关联起来的（`sort_order_id`）。
- 记住浮点排序的完整顺序。

## 核心概念

### 1. 结构

一个 sort order 由 **sort order id** + **sort fields 列表**组成，列表顺序就是排序优先级。每个 sort field 含：

| 组成 | 取值 |
|------|------|
| source column id / source column ids | 来自 table schema（v3 起支持多参数 transform 的 source ids） |
| transform | 与 partition transform **同一套**（`identity`、`bucket[N]`、`truncate[W]`、`year`、`day`…） |
| sort direction | 只能是 `asc` / `desc` |
| null order | 只能是 `nulls-first` / `nulls-last` |

### 2. 关键规则

- **order id `0` 保留给 unsorted**。
- 文件与 sort order 的关联方式是 manifest 里的 data file 字段 `sort_order_id`：

  - 只能给 data file 和 equality delete file 写非 null 的 `sort_order_id`；
  - **position delete file 必须设为 null**（它自己按 `file_path` + `pos` 排序，reader 必须忽略它的 `sort_order_id`）；
  - 缺失或未知的 sort order id ⇒ 当作 unsorted。
- 表有 `default-sort-order-id`，writer **应该**按它排序写入，但**不强制**（例如 streaming 写入时排序代价过高，可以不排）。
- 表必须在 metadata 的 `sort-orders` 列表里声明所有 sort order（供查找）。每次修改产生新的 sort order id，并更新 `default-sort-order-id`。

### 3. 浮点数的排序顺序

```
-NaN < -Infinity < -value < -0 < 0 < value < Infinity < NaN
```

这与 Java 浮点比较的实现一致。注意：

- `-0` 与 `0` 是有序的两个不同位置。
- bounds 的规则与之呼应：`-0.0` 必须排在 `+0.0` 之前，且 **NaN 不允许作为 lower/upper bound**（Day 09）。

## 自测题

1. `sort-order-id = 0` 表示什么？
2. position delete file 的 `sort_order_id` 应该写什么？为什么？
3. 表声明了 default sort order，writer 一定要按它排序吗？
4. 一个 sort field 用 `bucket[16]` 作为 transform 有意义吗？
5. `-0.0` 和 `0.0` 在 Iceberg 排序里是同一个位置吗？

<details>
<summary>参考答案</summary>

1. unsorted（无排序）。
2. 必须为 null；position delete file 自身要求按 `file_path` 再按 `pos` 排序，与表的 sort order 无关，reader 必须忽略该字段。
3. 不必须。规范说的是 should——因为排序可能非常昂贵（如流式写入）。
4. 有意义但少见：排序依据变成桶编号而非原始值，通常用于希望把同桶数据聚在一起、又不想按值排序的场景。
5. 不是。`-0 < 0`，两者是可区分的相邻位置。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/SortOrder.java`、`SortOrderBuilder.java`、`UnboundSortOrder.java`
- `core/src/main/java/org/apache/iceberg/SortOrderParser.java` — JSON 序列化
- `core/src/main/java/org/apache/iceberg/ReplaceSortOrder.java` — `ALTER TABLE … WRITE ORDERED BY` 对应的更新
- `data/src/main/java/org/apache/iceberg/data/` 下的排序读取工具（如 `BaseTaskWriter` 的分发逻辑）

## 一句话总结

**Sort order 用「source ids + transform + direction + null order」描述文件内的排序，通过 manifest 的 `sort_order_id` 挂在文件上，order id 0 表示未排序，而 position delete file 永远忽略它。**
