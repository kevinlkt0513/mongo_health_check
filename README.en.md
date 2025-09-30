## MongoDB Health Check & Design Review Tool

CLI tool to analyze MongoDB schema, indexes, document sizes, arrays, nesting, workload hints, and produce actionable recommendations.

### Features
- Schema sampling: field presence/types, polymorphism, arrays (max/p95/avg), nesting depth
- Document size distribution; warn near 16MB limit
- `collStats` and index sizes; largest index per collection
- Workload-based URI guidance using `serverStatus().opcounters` (when permitted)
- Reports: JSON + Markdown (EN/zh-TW) + DOCX (EN/zh-TW)

### Install
```bash
python -m venv .venv && .venv/ScriptS/pip install -r requirements.txt
```

### Quick Start

Use environment variables or `.env` to avoid exposing secrets in public repos.

1) Environment variable:
```bash
export MONGODB_URI="mongodb+srv://<username>:<password>@<cluster-url>/?retryWrites=true&w=majority&appName=<appName>"
python scripts/mongo_health_check.py --dbs appdb --collections orders --sample-size 200 --max-docs-per-coll 5000 --output-dir report
```

2) `.env` file:
- Copy `env.example` to `.env` and fill `MONGODB_URI`.
- Run:
```bash
python scripts/mongo_health_check.py --dbs appdb --collections orders --sample-size 200 --max-docs-per-coll 5000 --output-dir report
```

3) Custom env file path:
```bash
python scripts/mongo_health_check.py --env-file ".env.local" --output-dir report
```

Options:
- `--dbs db1,db2` limit databases
- `--collections c1,c2` limit collections (with --dbs)
- `--doc-id <id>` analyze a single document by `_id`
- `--filter '{"status":"A"}'` JSON filter for sampling
- `--timeout-ms`, `--seed`

Outputs:
- `report/report.json`, `report/report_en.md`, `report/report_zh-TW.md`
- `report/report_en.docx`, `report/report_zh-TW.docx`

Docs:
- Requirements (EN): `docs/requirements_tracking_en.md`
- Architecture (EN): `docs/architecture_overview_en.md`

