# ADR-0001：部署平台與提醒排程方式

- **狀態**：提議中（Proposed），等我確認後改為「已接受（Accepted）」
- **日期**：2026-10-21
- **決策者**：我
- **相關**：[plan.md](../plan.md)

> 這份是 Claude 寫的範例，用來示範 ADR 的格式和思考方式。從 ADR-0002 開始由我自己寫。

## 背景

習慣打卡 App 需要：

1. 一個 Go 後端 API 和一個 PostgreSQL 資料庫
2. **條件式提醒**：每個使用者設定一個時間，到時還沒打卡才發 Web Push
3. **每週小組總結**

限制條件：

- 預算每月 NT$300 以內，以免費額度為主
- 使用者只有 2 人，都在台灣，大部分時間沒有流量
- 這是學習專案，選擇要能接近業界主流的雲端架構

## 考慮過的選項

### 選項 A：Cloud Run + Neon + Cloud Scheduler

- Cloud Run 沒有流量時縮到 0 台，免費額度是每月 180,000 vCPU-秒、360,000 GiB-秒、200 萬次請求（request-based 計費）
- Neon 是 serverless PostgreSQL，閒置 5 分鐘後自動暫停，暫停期間不計費。免費方案每個專案每月 100 CU-hours
- Cloud Scheduler 每個帳單帳戶每月有 3 個免費 job，之後每個 job 每月 US$0.10
- **代價**：服務會睡著，所以**不能在程式裡自己跑排程**，必須由外部觸發；冷啟動會讓第一個請求變慢

### 選項 B：Railway 或 Fly.io 一直開著的小主機

- 程式裡直接跑一個 ticker 就能排程，最簡單
- 大約每月 US$5，在預算內
- **代價**：學不到「服務會縮到 0 台」這個雲端架構的核心限制，DB 還是要另外找

### 選項 C：自己租 VPS

- 最便宜、控制最多
- **代價**：要自己管 OS 更新、備份、TLS、安全性，時間會被維運吃掉，偏離「練架構決策」的目標

## 決定

**採用選項 A**，並且做以下設計：

### 1. 排程由外部觸發

- Cloud Scheduler 建立 **2 個 job**（在 3 個免費額度內）：
  - `reminders-tick`：每 15 分鐘呼叫一次 `POST /internal/jobs/reminders`
  - `weekly-summary`：每週日 20:00（Asia/Taipei）呼叫一次 `POST /internal/jobs/weekly-summary`
- 這兩個 endpoint 只接受 Cloud Scheduler 帶 OIDC token 的請求，後端要驗證 token，不能公開呼叫

### 2. 提醒時間以 15 分鐘為單位

這是這份 ADR 最重要的取捨，原因如下：

> **關鍵發現：排程每觸發一次，就會把 Neon 叫醒。**
> Neon 閒置 5 分鐘才會暫停。如果排程每 5 分鐘觸發一次，DB 就永遠不會暫停：
> 24 小時 × 30 天 × 0.25 CU（最小規格）= **180 CU-hours，超過免費的 100**。

改成每 15 分鐘觸發一次的話，每次觸發大約讓 DB 醒著 5 分鐘，也就是 1/3 的時間：
24 × 30 × 1/3 × 0.25 ≈ **60 CU-hours**，再加上兩個人實際使用的時間，仍然在 100 以內。

所以使用者設定提醒時間時，只能選 15 分鐘的倍數，例如 20:00、20:15。對習慣提醒來說，這個精度已經足夠。

### 3. 避免重複發送

每次觸發時，查詢「提醒時間落在這 15 分鐘內、今天（依使用者時區）還沒打卡、也還沒發過提醒」的使用者。

用一張 `reminder_sends` 表記錄已經發過的提醒，在 `(user_id, habit_id, local_date)` 上加 **unique constraint**。Cloud Scheduler 重試、或兩次觸發重疊時，重複的寫入會被資料庫擋下來，不會重複發送。

### 4. 區域

- Cloud Run 放在 `asia-east1`（台灣），離使用者最近
- Neon 選它在亞洲最近的區域（預計是新加坡，建立專案時要確認實際可選的區域）
- **代價**：API 每次查詢 DB 都要跨一段距離，大約多出數十毫秒。寫 API 時要減少每個請求查詢 DB 的次數。第 9～10 週做壓力測試時，再回來檢查這個決定

## 後果

**好處**

- 平常幾乎是 0 元，只有超出免費額度時才付費
- 會實際碰到 serverless 架構的限制：冷啟動、外部排程、冪等性、跨區延遲
- 排程邏輯集中在一個 endpoint，好測試，也可以手動觸發來除錯

**壞處與風險**

- 冷啟動：服務和 DB 都睡著時，第一個請求可能要等 1～數秒
- 提醒時間只能選 15 分鐘的倍數
- 外部排程如果沒有觸發，提醒就不會發出，需要靠 Sentry 和 log 才查得到
- 免費額度可能變動，Neon 自己的文件對免費方案的數字就有不一致的地方

**什麼情況下要重新檢討這個決定**

- Neon 的 CU-hours 用量接近 100
- 使用者抱怨提醒不夠準時
- 第 7～8 週拆出通知服務時，排程可能改成由 Cloud Tasks 為每個提醒建立精確時間的任務

## 待辦

- [ ] 建立 Neon 專案時，確認可選的亞洲區域和目前的免費額度
- [ ] 設定 GCP 預算警示（例如 US$5），避免意外產生費用
- [ ] 第 4 週上線一週後，到 Neon 後台確認實際的 CU-hours 用量，回來更新這份 ADR

## 參考資料（2026-10-21 查詢）

- [Cloud Run pricing](https://cloud.google.com/run/pricing)
- [Cloud Scheduler pricing](https://cloud.google.com/scheduler/pricing)
- [Neon plans](https://neon.com/docs/introduction/plans)
- [Neon pricing](https://neon.com/pricing)
