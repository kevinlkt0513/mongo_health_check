## MongoDB 健康檢查與設計評估工具

命令列工具，用於分析 MongoDB 的 Schema、索引、文件大小、陣列、巢狀深度、工作負載線索，並產出可執行建議。

### 功能
- Schema 採樣：欄位出現/型別、多態、陣列（max/p95/avg）、巢狀深度
- 文件大小分布：接近 16MB 上限給出警示
- `collStats` 與索引大小；每集合最大索引
- 基於 `serverStatus().opcounters` 的 URI 建議（在權限允許時）
- 報告：JSON + Markdown（英/繁）+ DOCX（英/繁）

### 安裝
```bash
python -m venv .venv && .venv/ScriptS/pip install -r requirements.txt
```

### 快速開始
```bash
python scripts/mongo_health_check.py --uri "<YOUR_MONGODB_URI>" --dbs appdb --collections orders --sample-size 200 --max-docs-per-coll 5000 --output-dir report
```

常用參數：
- `--dbs db1,db2` 限定資料庫
- `--collections c1,c2` 限定集合（需搭配 --dbs）
- `--doc-id <id>` 以 `_id` 指定單一文件
- `--filter '{"status":"A"}'` 以 JSON 條件進行採樣
- `--timeout-ms`、`--seed`

輸出：
- `report/report.json`、`report/report_en.md`、`report/report_zh-TW.md`
- `report/report_en.docx`、`report/report_zh-TW.docx`

文件：
- 需求（繁體）：`docs/requirements_tracking_zh-TW.md`
- 架構（繁體）：`docs/architecture_overview_zh-TW.md`

