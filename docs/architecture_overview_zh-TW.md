### MongoDB 健康檢查與設計評估 — 架構與規則

本文說明 CLI 工具之架構、資料流程、偵測規則、閾值與擴充性，供審閱參考。

---

## 1. 目標與非功能

- 只讀、易於命令列運行
- Python 3.9+，依賴精簡（pymongo、dnspython、python-docx）
- 權限受限時優雅降級
- 以採樣與上限保護線上環境

---

## 2. 輸入與範圍

- `--uri` 必填
- `--dbs`、`--collections` 範圍限制（預設排除系統 DB 與 `system.*`）
- `--doc-id`、`--filter` 目標化採樣
- `--sample-size`、`--max-docs-per-coll`、`--timeout-ms`、`--output-dir`

---

## 3. 資料流程

1) 連線探活；可得時抓取 `buildInfo`、`hello|isMaster`、`serverStatus`
2) 目標列舉（排除系統）
3) `collStats` 與 `estimated_document_count`
4) 採樣：`$sample` 或帶 `find` 過濾
5) 樣本分析：欄位出現/型別、陣列長度、BSON 大小分布、巢狀深度、近似基數
6) 規則→旗標→建議；索引洞見（高/低基數）
7) 報告：JSON、Markdown（英/繁）、DOCX（英/繁）

---

## 4. 規則與閾值

- 文件大小：`max ≥ 12MB` 告警；建議 Split/Reference/Bucket
- 巢狀深度：`max > 10` 告警；扁平化並僅內嵌熱門欄位
- 無上界陣列：`max ≥ 1000` 或 `p95 ≥ 500`；Bucket/Subset/Outlier
- 多態欄位：同一路徑多型別；統一或依變體拆分
- 基數（近似）：每欄位去重上限 5000；輔助索引排序

---

## 5. 建議對應

- 大型文件 → 拆分/引用/分桶
- 深度巢狀 → 扁平化；僅內嵌熱門欄位
- 無上界陣列 → 分桶/子集/異常外移；注意多鍵索引
- 多態 → 統一型別或拆分
- 索引 → 高基數做前導過濾；低基數避免前導

---

## 6. URI 與負載建議

依 `serverStatus().opcounters` 推導讀寫比；建議 `readPreference`、`retryWrites`、`writeConcern` 與連線池大小。

---

## 7. 輸出

- JSON：server、collections（collStats、schema、flags、indexInsights、recommendations）、uriRecommendations、errors
- Markdown：按資料庫分組的彙總表 + 集合級 Schema/Index 建議表 + URI 小節（含指標）
- DOCX：雙語版本

---

## 8. 擴充

- 閾值參數化
- 整合 `$indexStats`（未用索引）、慢查詢/Profiler 訊號
- 時序/TTL/分片鍵啟發式、寫入熱點檢查


