### MongoDB Health Check & Design Review — Architecture & Rules

This document explains the CLI tool architecture, data flow, detection rules, thresholds, and extensibility for reviewers.

---

## 1. Goals & NFRs

- Read-only, CLI-friendly health check and design review
- Python 3.9+, minimal dependencies (pymongo, dnspython, python-docx)
- Graceful degradation when commands are restricted
- Sampling caps to protect production workloads

---

## 2. Inputs & Scoping

- `--uri` MongoDB URI (required)
- `--dbs`, `--collections` to limit scope (system DBs and `system.*` excluded by default)
- `--doc-id` and `--filter` for targeted sampling
- `--sample-size`, `--max-docs-per-coll`, `--timeout-ms`, `--output-dir`

---

## 3. Data Flow

1) Connect & probe (ping); fetch `buildInfo`, `hello|isMaster`, `serverStatus` if allowed
2) Enumerate target DBs/collections (exclude system)
3) `collStats` + `estimated_document_count`
4) Sampling via `$sample` or `find` with filters
5) Sample analysis: field presence/types, array length stats, BSON size dist, nesting depth, approximate cardinality
6) Rules → flags → recommendations; index insights (high/low cardinality)
7) Reports: JSON, Markdown (EN/zh-TW), DOCX (EN/zh-TW)

---

## 4. Rules & Thresholds

- Document size: warn when max ≥ 12MB; recommend Split/Reference/Bucket
- Nesting depth: warn when max > 10; flatten and embed hot fields only
- Unbounded arrays: risk when max ≥ 1000 or p95 ≥ 500; Bucket/Subset/Outlier
- Polymorphism: multiple types for same path; normalize or split by variant
- Cardinality (approx.): distinct capped at 5000 per field; guide index order

---

## 5. Recommendations Mapping

- Large documents → split/reference/bucket patterns
- Deep nesting → flatten; embed hot fields
- Unbounded arrays → bucket/subset/outlier; beware multikey
- Polymorphism → normalize types or split
- Indexes → high-cardinality as leading filter keys; avoid low-cardinality lead

---

## 6. URI & Workload Guidance

From `serverStatus().opcounters` derive read/write ratio; suggest `readPreference`, `retryWrites`, `writeConcern`, and pool sizing appropriately.

---

## 7. Output Structure

- JSON: server, collections (collStats, schema, flags, indexInsights, recommendations), uriRecommendations, errors
- Markdown: per-DB grouped summary + per-collection tables (schema/index) + URI section with evidence
- DOCX: bilingual equivalents

---

## 8. Extensibility

- Parameterize thresholds
- Integrate `$indexStats` (unused indexes), slow query/profiler signals
- Time series/TTL/sharding key heuristics, write hot-spot checks


