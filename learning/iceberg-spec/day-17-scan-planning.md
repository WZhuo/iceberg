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

# Day 17 · Scan Planning：裁剪与去重规则

> **阅读材料**：`format/spec.md` 第 1047–1093 行（Scan Planning）
> **预计用时**：40 分钟
> **前置**：Day 05、Day 09、Day 11、Day 14

## 今日目标

- 分清 inclusive projection 与 strict projection（residual predicate）。
- 能完整背出 DV / position delete / equality delete 的适用条件。
- 理解「文件路径在 snapshot 内最多出现一次」这条不变式。

## 核心概念

### 1. plan 的目标

从「用户的谓词」出发，用**最少**的元数据读出「需要扫描的文件集合」：

```
table metadata → snapshot → manifest list（按 partition summary 跳 manifest）
              → manifest（按 metrics 跳 data file / delete file）
              → 得到 scan tasks（data file + 需要应用的 deletes）
```

### 2. 两类 partition 投影

| 投影 | 含义 | 用途 |
|------|------|------|
| **inclusive projection** | 生成的分区谓词会「选中可能包含匹配行的文件」 | 选择要读的文件 |
| **strict projection** | 生成的分区谓词会「选中所有行必然匹配的文件」 | 计算每个文件的 **residual predicate**（剩余谓词） |

原文例子：`ts > X` 对 `ts_day = day(ts)` 的 inclusive projection 是 `ts_day >= day(X)`——注意这通常会**多包含**一点范围（因为 `X` 之前一小段时间与 `X` 之后的落在同一天），所以还需要 residual predicate 再过滤。

- **未知 transform 的 inclusive projection 恒为 true**（该分区字段被忽略）。

### 3. metrics 过滤对 data file 与 delete file 是同一套

因为 manifests 里 data file 与 delete file 存同样的统计信息（Day 09），所以：

> 如果一个 delete file 的 metrics 表明它不可能有行匹配扫描谓词，就可以像忽略 data file 一样忽略这个 delete file。

### 4. Delete 的适用范围（核心表，务必记牢）

| 载体 | 适用条件 |
|------|----------|
| **Deletion vector** | ① data file 的 `file_path` == DV 的 `referenced_data_file`；② data file 的 data sequence number **≤** DV 的 data sequence number；③ partition（spec 与取值）相等 |
| **Position delete file** | ① 若其 `referenced_data_file` 非空，必须等于 data file 路径；② data sequence number **≤** delete 的；③ partition 相等；④ **该 data file 没有必须应用的 DV**（有 DV 时会包含所有历史 position delete） |
| **Equality delete file** | ① data file 的 data sequence number **严格小于** delete 的；② partition 相等，**或者** delete file 使用 unpartitioned spec |

两个特殊情形：

- **unpartitioned spec 的 equality delete file 是全局删除**，可作用于任意分区的 data file。
- **position delete（DV 与文件）可以作用于同一次提交中加入的行**（sequence number 相等也成立），这样才能「同一 commit 里先插入再删除」。

### 5. 其他 plan 规则

- **文件路径在同一个 snapshot 内最多出现一次**（跨所有 manifest 的 ADDED / EXISTING 聚合）。若重复出现，扫描结果未定义；reader 可以报错（不是必须）。
- **partition 相等性判定**：
  - 浮点分区值按 IEEE 754 位模式比较（Java 里等价于 `Float.floatToIntBits` / `Double.doubleToLongBits`），NaN 归一化为只保留最高尾数位。
  - **未知 transform 不影响分区相等性**：过滤时忽略该字段，但判断「partition 是否相等」时仍要用未知 transform 算出的值。

## 自测题

1. inclusive projection 与 strict projection 的输出分别用来做什么？
2. 为什么 `ts > X` 生成的 inclusive projection 是 `ts_day >= day(X)` 而不是 `=`？
3. 什么时候可以忽略一个 delete file？
4. 什么情况下 position delete 可以删掉与自己 sequence number **相同**的 data file 中的行？
5. 同一 snapshot 的两个 manifest 里出现同一个 data file 路径，会怎样？

<details>
<summary>参考答案</summary>

1. inclusive 用来选文件（可能匹配即保留）；strict 用来算 residual predicate（必然匹配的部分可以直接下推掉）。
2. 因为 `X` 所在那一天里既有 `< X` 的行也有 `> X` 的行，`day(X)` 这一天的文件都可能命中，必须包含进来，再由 residual predicate 精确过滤。
3. 当 delete file 的 metrics（counts / bounds）表明它不可能包含匹配扫描谓词的行时。
4. 同一次提交内既插入又删除时（position delete 的条件是 `≤`，不是严格小于）。
5. 扫描结果未定义；实现可以选择抛错。规范要求同一个 snapshot 内每个文件路径只出现一次。

</details>

## 与代码对应

- `api/src/main/java/org/apache/iceberg/expressions/` — `InclusiveMetricsEvaluator`、`StrictMetricsEvaluator`、`InclusiveProjection`、`StrictProjection`
- `core/src/main/java/org/apache/iceberg/ManifestGroup.java` — 组合 data 与 delete 文件、应用 sequence number 规则
- `core/src/main/java/org/apache/iceberg/DataTableScan.java`、`SnapshotScan.java`、`TableScanContext.java` — scan 的构建与 refinement（注意每次 refinement 必须是独立的新 scan）
- `data/src/main/java/org/apache/iceberg/data/DeleteFilter.java` — 读时合并删除

## 一句话总结

**Plan = 用 partition summary 跳 manifest、用 metrics 跳文件、再按「partition 相等 + sequence number 大小」把 delete 应用到正确的 data file 上；inclusive 选文件，strict 算 residual。**
