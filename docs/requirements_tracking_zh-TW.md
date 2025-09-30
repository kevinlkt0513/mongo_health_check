### 需求追蹤文件：MongoDB 健康檢查與設計優化

本文件用於追蹤 CLI 工具之需求、範圍、交付物、檢查清單與改進建議，便於審閱與後續維護。

---

## 一、背景與目標

對指定集群/資料庫/集合進行可重複、可自動化的健康檢查與設計評估，涵蓋 Schema、索引、文件尺寸、陣列模式、巢狀深度、工作負載與連線參數建議等，輸出 JSON 與 Markdown 報告並提供可行建議。

---

## 二、輸入與執行

- 必填：MongoDB URI
- 選填：
  - `--dbs db1,db2` 僅分析指定資料庫（預設排除 `admin/local/config` 與 `system.*`）
  - `--collections c1,c2` 僅分析所列資料庫中的集合
  - `--sample-size` 每集合採樣數（預設 200）
  - `--max-docs-per-coll` 每集合掃描上限（預設 5000）
  - `--output-dir` 輸出資料夾（預設 `report/`）
  - `--timeout-ms` 連線/命令逾時
  - `--doc-id` 僅分析指定 `_id` 之文件（支援 ObjectId 與字串）
  - `--filter` 以 JSON 篩選條件進行目標化採樣

---

## 三、輸出與交付物

- 報告：
  - JSON：`report/report.json`
  - Markdown：`report/report.md`、`report/report_en.md`、`report/report_zh-TW.md`
  - DOCX：`report/report_en.docx`、`report/report_zh-TW.docx`
- 終端摘要：重點指標與告警

涵蓋：
1. 叢集/伺服器資訊（在權限允許時）
2. 索引清單與大小；每集合最大索引
3. Schema 採樣：欄位出現率/型別、多態、陣列長度分布、文件大小分布、巢狀深度
4. 工作負載（若可取得 `opcounters`）與 URI 調校建議
5. 依最佳實務提出每集合改進建議

---

## 四、偵測規則（初版）

- 文件大小：
  - `max ≥ 12MB` 告警（接近 16MB 上限）；建議 Split/Reference/Bucket
- 陣列：
  - `max ≥ 1000` 或 `p95 ≥ 500` 判定無上界風險；建議 Bucket/Subset/Outlier
- 多態欄位：
  - 同一路徑多種型別；建議統一型別或依變體拆分
- 巢狀深度：
  - `maxDepth > 10` 告警；建議扁平化、僅內嵌熱門欄位
- 索引：
  - 列示最大索引；鼓勵以 `$indexStats`/計劃驗證
- 工作負載：
  - 以 `serverStatus().opcounters` 估算讀寫比；調整 readPreference、retryWrites、writeConcern、連線池等

---

## 五、非功能性需求

- 友善 CLI；Python 3.9+
- 僅讀取操作；權限不足時優雅降級並標註
- 以採樣與上限保護線上環境
- 輸出結構清楚，便於審查

---

## 六、里程碑與狀態

1. 需求整理（本文）— 已完成
2. CLI 工具（統計、採樣、Schema 推斷、索引大小、報告）— 已完成（持續優化）
3. 基於工作負載的 URI 建議 — 已完成（持續優化）
4. 文件與使用說明 — 已完成

---

## 七、風險與假設

- 權限可能限制 `serverStatus`、`$indexStats` 等
- 採樣代表性有限；建議在低峰或影子環境執行
- 基數估計為樣本近似值（有上限）

---

## 八、參考

- MongoDB Support Tools（概念參考）


