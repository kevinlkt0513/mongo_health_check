### Requirements Tracking: MongoDB Health Check & Design Optimization

This document tracks the requirements, scope, deliverables, checklists, and improvement items for the CLI-based MongoDB health check and design review tool.

---

## 1. Background & Goals

Provide a repeatable and automated health check and design review across schema, indexes, document size, arrays, nesting depth, workload signals, and connection string recommendations. Input is a MongoDB URI with optional DB/collection scope; output includes JSON and Markdown reports with actionable guidance based on MongoDB best practices.

---

## 2. Inputs & Execution

- Required: MongoDB URI
- Optional:
  - `--dbs db1,db2` limit to specific databases (system DBs and `system.*` collections are excluded by default)
  - `--collections c1,c2` limit to collections within provided DBs
  - `--sample-size` per-collection sample size (default 200)
  - `--max-docs-per-coll` per-collection scan cap (default 5000)
  - `--output-dir` output directory (default `report/`)
  - `--timeout-ms` connection/command timeout
  - `--doc-id` sample only a specific document by `_id` (ObjectId or string)
  - `--filter` JSON filter for targeted sampling

---

## 3. Outputs & Deliverables

- Reports:
  - JSON: `report/report.json`
  - Markdown: `report/report.md`, `report/report_en.md`, `report/report_zh-TW.md`
  - DOCX: `report/report_en.docx`, `report/report_zh-TW.docx`
- Console: summary highlights and warnings

Coverage:
1. Cluster/server info (when permitted)
2. Index inventory and sizes; largest index per collection
3. Schema sampling: field presence/types, arrays (length dist), polymorphism, document size dist, nesting depth
4. Workload (opcounters when permitted) and URI tuning suggestions
5. Best-practice alignment recommendations per collection

---

## 4. Detection Rules (Initial)

- Document size:
  - Warn when max ≥ 12MB (close to 16MB limit). Recommend Split/Reference/Bucket patterns
- Arrays:
  - Unbounded risk when max ≥ 1000 or p95 ≥ 500. Recommend Bucket/Subset/Outlier patterns
- Polymorphism:
  - Same field with multiple types. Recommend normalization or schema variant split
- Nesting depth:
  - Warn when max depth > 10. Recommend flattening; embed only hot fields
- Indexes:
  - List largest index; where allowed, encourage validation via `$indexStats`/plans
- Workload:
  - From `serverStatus().opcounters` estimate read/write ratio; align URI readPreference, retryWrites, writeConcern, and pool sizing

---

## 5. Non-Functional Requirements

- CLI-friendly; Python 3.9+
- Read-only; graceful degradation under limited privileges
- Sampling caps to protect production
- Clear, structured outputs for reviews

---

## 6. Milestones & Status

1. Requirements tracking (this doc) — completed
2. CLI tool (stats, sampling, schema inference, index sizes, reports) — completed (iterative)
3. URI recommendations based on workload — completed (iterative)
4. Documentation & usage — completed

---

## 7. Risks & Assumptions

- Privileges may restrict `serverStatus`, `$indexStats`, etc.
- Sampling representativeness; advisable to run off-peak or in a shadow env
- Cardinality is approximate (sample-based, capped)

---

## 8. References (Design Guidance)

- MongoDB Support Tools (conceptual reference)


