## MongoDB 健康检查与设计评估工具

本仓库提供一个可在命令行运行的 Python 脚本，用于对 MongoDB 集群/数据库/集合进行健康检查与设计评估：
- Schema 推断、字段类型分布与多态检测
- 数组长度分布与 Unbounded Array 风险
- 文档大小分布，提示接近/超过 16MB 风险
- 嵌套深度估算
- `collStats` 与索引大小，输出最大索引
- 基于 `serverStatus().opcounters` 的读写比例估算（若权限允许）
- 基于工作负载与最佳实践给出连接字符串优化建议

参考资料：MongoDB Support Tools（设计与实现思路参考） `https://github.com/mongodb/support-tools`

### 安装依赖

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Windows PowerShell 可使用：

```powershell
python -m venv .venv; .\.venv\Scripts\Activate.ps1; pip install -r requirements.txt
```

### 使用示例

推荐使用环境变量或 `.env`，避免在公开仓库暴露敏感信息：

1) 直接使用环境变量（示例）：
```bash
set MONGODB_URI="mongodb+srv://<username>:<password>@<cluster-url>/?retryWrites=true&w=majority&appName=<appName>"
python scripts/mongo_health_check.py --sample-size 200 --max-docs-per-coll 5000 --output-dir report
```

2) 使用 `.env` 文件（推荐本地开发）：
- 复制根目录的 `env.example` 为 `.env`，并填写 `MONGODB_URI`
- 运行：
```bash
python scripts/mongo_health_check.py --output-dir report
```

3) 指定自定义 `.env` 路径：
```bash
python scripts/mongo_health_check.py --env-file ".env.local" --output-dir report
```

可选参数：
- `--dbs db1,db2` 仅分析指定数据库
- `--collections coll1,coll2` 仅分析指定集合（需配合 `--dbs`）
- `--timeout-ms 10000` 连接超时
- `--seed 42` 采样随机种子
- `--doc-id <id>` 仅分析匹配 `_id` 的单文档（自动尝试 ObjectId）
- `--filter '{"status":"A"}'` 基于 JSON 过滤条件采样分析

输出：
- `report/report.json` 全量结构化报告
- `report/report.md` Markdown 摘要
- `report/report_en.docx` 英文版 DOCX（含表格）
- `report/report_zh-TW.docx` 繁體中文版 DOCX（含表格）

### 文档资料（Docs）

- Requirements（中文简体）: [docs/requirements_tracking.md](docs/requirements_tracking.md)
- Requirements（English）: [docs/requirements_tracking_en.md](docs/requirements_tracking_en.md)
- Requirements（繁體）: [docs/requirements_tracking_zh-TW.md](docs/requirements_tracking_zh-TW.md)
- Architecture（中文简体）: [docs/architecture_overview.md](docs/architecture_overview.md)
- Architecture（English）: [docs/architecture_overview_en.md](docs/architecture_overview_en.md)
- Architecture（繁體）: [docs/architecture_overview_zh-TW.md](docs/architecture_overview_zh-TW.md)

### 快速开始（Quick Start）

- English README: [README.en.md](README.en.md)
- 繁體 README: [README.zh-TW.md](README.zh-TW.md)

English sample run:
```bash
python scripts/mongo_health_check.py --uri "<YOUR_MONGODB_URI>" --dbs appdb --collections orders --sample-size 200 --max-docs-per-coll 5000 --output-dir report
```

### 安全性

脚本仅执行只读操作（stats、aggregate $sample、find limit）。在权限不足时自动降级并记录说明。

为避免凭证泄露：
- 不要将真实连接字符串写入 README、代码或提交历史
- 使用 `MONGODB_URI` 环境变量或 `.env` 文件（`.gitignore` 已忽略 `.env`）
- 在 CI/CD 中使用仓库机密变量（如 GitHub Actions Secrets）

### 许可

仅供内部评估使用。参考开源工具（思路）请见：`https://github.com/mongodb/support-tools`


