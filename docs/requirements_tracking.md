### 需求跟踪文档：MongoDB 健康检查与设计优化工具

本文件用于跟踪“MongoDB 多功能健康检查与设计评估”工具的需求、范围、交付物、检查清单与改进建议。

---

## 一、背景与目标

客户希望基于现有 MongoDB 实例，进行一次可重复的、可自动化的健康检查与设计评估，覆盖 Schema、索引、文档尺寸、数组模式、读写工作负载、连接参数优化等方面，并生成结构化报告（JSON/Markdown）。

工具需以命令行方式运行，输入 MongoDB 连接字符串（URI），可选限定 DB/Collection 范围，输出分析结果与改进建议，符合 MongoDB 最佳实践。可参考 MongoDB 官方支持工具的思路与实现范式，以便进一步扩展与自动化维护（参考链接见下）。

参考资料：
- MongoDB Support Tools（参考设计与思路）: `https://github.com/mongodb/support-tools`

---

## 二、输入与运行方式

- 必填：MongoDB 连接字符串（URI），例如：
  - `mongodb+srv://new-user-01:new-user-01@clusterm10.4y4hg.mongodb.net/?retryWrites=true&w=majority&appName=ClusterM10`
- 可选：
  - 限定数据库：`--dbs db1,db2`
  - 限定集合：`--collections coll1,coll2`（若提供 `--dbs` 与 `--collections`，则对指定 DB 的指定集合生效）
  - 采样数量：`--sample-size`（每个集合随机/顺序采样文档数量，用于 Schema 推断与模式分析）
  - 每集合最大扫描文档数上限：`--max-docs-per-coll`
  - 输出目录：`--output-dir`（默认 `report/`）
  - 超时：`--timeout-ms`（连接与操作级别）

---

## 三、输出与交付物

- 结构化报告
  - JSON：`report/report.json`
  - Markdown：`report/report.md`
- 控制台摘要：关键结论与告警、Top 级指标

报告需覆盖：
1. 基础信息：
   - Cluster/Server 版本、部署拓扑（可获取范围内）
   - 可见的数据库、集合、文档数、存储大小、平均文档大小
2. 索引：
   - 每集合索引列表与大小（`collStats.indexSizes`）
   - 最大索引的名称与大小
   - `$indexStats` 使用统计（可用则输出）
3. Schema 与模式（基于采样/扫描）：
   - 字段类型分布、字段出现率
   - 多态字段（同字段多类型）
   - 数组字段：长度分布、最大长度、是否疑似 Unbounded Array
   - 嵌套深度（过深嵌套提醒）
   - 文档体积检测：接近/超过上限风险（16MB），建议分片/重构
4. 读写与连接（在权限允许时）：
   - 基于 `serverStatus().opcounters` 估算全局读写比例
   - 若可得，`$collStats`/latency 信息
   - 基于读写特征给出连接字符串优化建议（例如 `readPreference`、`retryWrites`、`w` 值、`maxPoolSize` 等）
5. 最佳实践对齐建议：
   - 针对 Unbounded Array、超大文档、过度多态、索引冗余或缺失等提出具体改进方向

---

## 四、检测规则（初版）

- 文档大小：
  - 计算 BSON 尺寸；≥ 12MB 警告，≥ 16MB 风险（上限），建议拆分（Bucket/分集合/引用模式）
- 数组：
  - 记录最大长度、分位数（如 P95）；超过阈值（默认 1000）或增长无上界迹象，标记 Unbounded Array 风险
- 多态：
  - 同字段出现多种类型（如 string/number/object 混用），按占比排序输出
- 嵌套深度：
  - 深度 > 10 给予提醒
- 索引：
  - 列出最大索引；若 `$indexStats` 可用则输出访问频率，识别长期未用索引
- 工作负载：
  - 基于 `opcounters` 粗略估算读写比；若写多读少，优先主节点读；若读多且可容忍延迟，可考虑 `secondaryPreferred`

---

## 五、非功能性要求

- 可在命令行直接运行，Python 3.9+，依赖尽量精简（`pymongo`, `dnspython`）
- 对权限不足/不可用命令需优雅降级并在报告标注原因
- 默认对每集合采样数量与扫描上限有限制，避免压测线上
- 运行安全：只做只读查询（stats、sample、find limit），不做写操作

---

## 六、里程碑与状态

1. 需求整理与跟踪文档（当前文件）
   - 状态：完成（初版）
2. 初版 CLI 工具实现（连接、采样、Schema 推断、索引、报告）
   - 状态：进行中
3. 连接参数优化建议完善（结合工作负载）
   - 状态：进行中
4. 文档与使用说明（README）
   - 状态：待完成

---

## 七、已知风险与假设

- 跨库/跨集合的统计可能受权限限制
- `$indexStats`、`serverStatus` 在受限环境不可用
- 样本代表性受限，建议在业务低峰期或使用影子环境运行

---

## 八、参考链接

- MongoDB Support Tools（思路参考）：`https://github.com/mongodb/support-tools`


