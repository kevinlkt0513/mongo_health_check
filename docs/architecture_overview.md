### MongoDB 健康检查与设计评估 — 架构与规则说明

本文件面向评审者，系统性说明 CLI 工具的总体架构、数据流、检测规则、阈值与可扩展点，便于 Review 与后续演进。

---

## 1. 目标与非功能要求

- **目标**：对指定集群/库/集合执行只读健康检查，产出结构化报告（JSON+Markdown），聚焦 Schema、索引、文档尺寸、数组模式、嵌套深度、工作负载与连接参数建议。
- **运行方式**：命令行执行，Python 3.9+，依赖精简（pymongo、dnspython）。
- **只读**：仅调用 stats/aggregate/find 等只读接口；权限不足自动降级并记录说明。
- **性能安全**：通过采样与最大扫描上限控制对线上影响。
- **可维护性**：规则阈值集中管理，可参数化扩展。

---

## 2. 输入与筛选

- 必填：`--uri` MongoDB 连接字符串。
- 选择性限制范围：
  - `--dbs db1,db2` 仅分析指定数据库（默认排除 `admin/local/config`）。
  - `--collections coll1,coll2` 仅分析指定集合（需配合 `--dbs`）。
  - `--doc-id <id>` 仅分析匹配 `_id` 的单文档（支持 ObjectId 与字符串）。
  - `--filter '<json>'` 为采样构建查询过滤条件（JSON 字符串）。
- 采样控制：
  - `--sample-size` 每集合采样文档数量（默认 200）。
  - `--max-docs-per-coll` 每集合扫描上限（默认 5000）。
  - 优先使用 `$sample`；若指定 `--filter/--doc-id` 则改用 `find` 保持语义可控。

---

## 3. 数据流与模块

1) 连接初始化与探活：`admin.command('ping')`；获取 `buildInfo/hello|isMaster/serverStatus`（可用时）。

2) 目标枚举：按 `--dbs/--collections` 过滤，并默认剔除系统 DB 与 `system.*` 集合。

3) 集合级统计：
   - `collStats`：`count/size/avgObjSize/storageSize/totalIndexSize/indexSizes`。
   - `estimated_document_count()`：文档量估计。

4) 采样：按上述策略收集样本集（文档列表）。

5) 样本分析：
   - 字段路径展平：记录字段出现次数 `fieldPresence`、类型分布 `fieldTypes`。
   - 数组统计：收集数组字段长度，计算 `max/p95/avg/samples`。
   - 文档大小：BSON 编码估算每文档字节数，汇总 `max/p95/avg`。
   - 嵌套深度：递归估算最大深度、分位数与均值。
   - 近似基数：对非 `array/object` 字段，截断去重（上限 5000）估算 distinct。

6) 规则判定与建议：
   - `flags`：基于统计设置风险标记（详见 §4）。
   - `recommendations`：将命中的规则翻译成具体建议（详见 §5）。
   - `indexInsights`：输出高/低基数字段候选，供索引设计参考。

7) 报告生成：
   - JSON：完整原始结构（含错误与说明）。
   - Markdown：按 DB 分组的集合汇总表 + 每集合的 Schema/Index 建议表。

---

## 4. 判定规则与阈值

- 文档大小（BSON）：
  - 预警阈值：`max >= 12MB` 视为大文档风险（接近 16MB 上限）。
  - 建议方向：拆分（Split）、引用（Reference）、分桶（Bucket）。

- 嵌套深度：
  - 告警阈值：`maxDepth > 10`。
  - 建议方向：扁平化结构，仅内嵌热点字段。

- Unbounded Array：
  - 统计：对数组字段计算 `max/p95/avg`。
  - 风险阈值：`max >= 1000` 或 `p95 >= 500` 视为无上界风险。
  - 建议方向：桶化（按时间/区间）、子集化（仅存近期/热点）、异常外移（Outlier）。

- 多态字段：
  - 判定：同一字段路径出现多种类型则标记为多态。
  - 建议方向：规范字段类型或拆分为不同变体集合。

- 字段基数与出现率：
  - 近似 distinct：对标量值进行去重（上限 5000），计算 `distinct/present`。
  - 高基数候选：`present >= max(30, 0.3 * sampleCount)` 且 `distinct >= 30` 且 `distinct/present >= 0.5`。
  - 低基数警示：`distinct <= 5` 且 `presenceRatio >= 0.5`，不建议作为复合索引前导位。
  - 大数组索引警示：数组长度很大时，提示多键索引条目爆炸风险。

> 注：阈值可参数化；默认值兼顾通用性与稳健性，Review 时可结合业务场景微调。

---

## 5. 建议生成（Recommendation Mapping）

- 大文档：
  - Evidence：`docSizeBytes.max/p95`。
  - Recommendation：Split/Reference/Bucket 模式。

- 深嵌套：
  - Evidence：`nestingDepth.max/p95`。
  - Recommendation：扁平化；内嵌仅保留高频访问字段。

- Unbounded Array：
  - Evidence：数组字段 `max/p95` 超阈值。
  - Recommendation：Bucket/Subset/Outlier；谨慎创建多键索引。

- 多态字段：
  - Evidence：`fieldTypes[path]` 出现多类型。
  - Recommendation：统一类型或拆分集合/版本化。

- 索引（基数驱动）：
  - 高基数候选：建议作为过滤条件时考虑进入索引前导位。
  - 低基数警示：避免作为复合索引前导位，放在后位或不建索引。
  - 大数组：警示多键索引条目数量爆炸，评估代价与收益。

---

## 6. 连接字符串与工作负载建议

- 从 `serverStatus().opcounters` 粗略估算读写比：读多（≥5:1）可考虑 `secondaryPreferred`（可接受延迟时）；写多（≤0.2:1）优先主节点读并确保 `retryWrites=true`。
- 建议明确 `writeConcern` 以匹配耐久性需求（如 `w=majority`）。
- `maxPoolSize` 随客户端并发与服务器能力调优。

> 注：工作负载建议为宏观指引；若需细粒度优化建议，可结合慢日志、`$indexStats` 与查询计划进一步分析。

---

## 7. 输出结构（摘要）

- JSON（`report/report.json`）：
  - `server`、`collections[]`（含 `collStats`、`schema`、`flags`、`indexInsights`、`recommendations`）、`uriRecommendations`、`errors`。
- Markdown（`report/report.md`）：
  - 按 DB 分组的集合汇总表（核心容量与告警列）。
  - 每集合两张表：`Schema Recommendations` 与 `Index Recommendations`（三列：Issue/Evidence/Recommendation 与 Category/Fields/Note）。

---

## 8. 限制与风险

- 权限限制：`serverStatus`、`$indexStats` 等可能不可用；脚本会降级并记录备注。
- 抽样代表性：基于采样推断，建议与业务侧二次确认；必要时扩大样本或离线分析。
- 近似基数：为样本级估计（带上限），不等价于全量 distinct；用于启发式索引建议。

---

## 9. 可扩展点（后续工作）

- 参数化阈值：将大小/深度/数组长度/基数阈值暴露为 CLI 参数与配置文件。
- `$indexStats` 集成：若权限允许，补充未使用索引清单与访问频率。
- 慢查询/命中率：结合 profiler/慢日志，补充查询模式与索引命中分析。
- 专项集合支持：Time Series、TTL、分片键评估、写放大热点识别等。


