# 分店 `visit_mode` 三種模式與程式策略

## 一、三種模式的業務邏輯

### 1) `DISABLED`（不追桌／櫃檯快點）
- **定位**：外帶、立食、早餐尖峰；**不建立 visit，不管座位**。  
- **掃碼入口**：直接顯示菜單（依 `menu_version` 命中規則）。  
- **下單關聯**：  
  - 每位客人各自一張 `order_info`，**沒有 `visit_id`**。  
  - 彼此不可見（靠 `customer_token`）。  
- **單人/多人同時點餐**：**可以**，互不影響。  
- **KDS 顯示**：以 **訂單** 為粒度（無桌聚合），可用「取餐號」。  
- **付款**：線上或臨櫃皆可。  
- **清桌**：無清桌概念。  
- **場景**：最簡單，適合高峰外帶。

---

### 2) `ATTENDED`（帶位制）
- **定位**：由店員控制座位。  
- **掃碼入口**：  
  - 店員在 POS 先開桌（建立 `visit.OPEN`）。  
  - 客人掃碼進入該 `visit` 下單；若桌未開，顯示「請店員帶位」。  
- **下單關聯**：  
  - 同桌共用一個 `visit_id`。  
  - 每人各自 `order_info`，KDS 以 `visit_id` 聚合。  
- **單人/多人同時點餐**：**可以**，同桌共用。  
- **KDS 顯示**：以桌為單位聚合顯示。  
- **付款**：各自付款，也可由 POS 代付。  
- **清桌**：  
  - 店員手動清桌（不得有未付款/進行中）。  
  - POS 可支援 **force 清桌**（取消未出餐訂單並回補配額）。  
- **場景**：內用為主，服務流程嚴謹。

---

### 3) `AUTO`（自動開桌／桌碼快翻台）
- **定位**：內用但希望快速翻桌、減少人工作業。  
- **掃碼入口**：  
  - **首位掃碼** → 自動建立 `visit.OPEN`，`table_info.status=IN_USE`。  
  - 後續掃同一桌碼 → 加入同一 `visit_id`。  
- **下單關聯**：  
  - 同桌各自 `order_info`，共享 `visit_id`。  
  - KDS 以桌聚合。  
- **單人/多人同時點餐**：**可以**。  
- **KDS 顯示**：按 `visit_id` 聚合。  
- **付款**：各自付款。  
- **清桌**：  
  - **自動**：最後互動時間 + `auto_close_minutes`，且無未付款/進行中 → 自動清桌。  
  - **手動**：POS 一鍵清桌，支援 force。  
  - **安全**：清桌時可重置 QR Token 避免舊客亂點。  
- **場景**：快翻桌早餐/午餐店。

---

## 二、差異一覽表

| 面向 | DISABLED | ATTENDED | AUTO |
|---|---|---|---|
| 是否建立 `visit` | 否 | 是（店員開） | 是（首掃自動開） |
| 同桌多人各自點餐 | 可以（不聚合） | 可以（聚合） | 可以（聚合） |
| KDS 聚合單位 | 訂單 | 桌 (`visit`) | 桌 (`visit`) |
| 清桌方式 | 不適用 | 手動（可 force） | 自動 + 手動（可 force） |
| 店員介入 | 最少 | 高（帶位/清桌） | 低（例外時） |
| 最佳場景 | 外帶尖峰 | 嚴格服務流程 | 快翻桌內用 |

---

## 三、程式策略差異

### 1. 掃碼入口
- **共通**：以 `qr_token` 找 `table_id`，命中菜單版本。  
- **分歧**：  
  - `DISABLED`：直接下單 session（無 visit）。  
  - `ATTENDED`：桌未開 → 回「請店員帶位」。  
  - `AUTO`：桌未開 → 新建 `visit.OPEN`。

---

### 2. `visit` 狀態與清桌
- `DISABLED`：無狀態機。  
- `ATTENDED`：手動清桌。  
- `AUTO`：需背景 Job 檢查 auto-close 條件。

---

### 3. 併發與鎖
- `AUTO`：首掃建桌需 Redis 分散鎖避免重複。  
- 共通：下單時靠 `(branch_id, client_txn_id)` 唯一鍵 + Redis Lua 配額控制。  
- 清桌需序列化操作（關閉 visit → 取消訂單 → 回補配額）。

---

### 4. QR Token 策略
- **共通**：`table_info.qr_token` 唯一。  
- `AUTO/ATTENDED`：清桌可重置 token。  
- `DISABLED`：可用門市級 QR。

---

### 5. KDS 聚合
- `DISABLED`：顯示單筆訂單。  
- `ATTENDED/AUTO`：依 `visit_id` 聚合顯示同桌。

---

### 6. 前台（顧客端）
- `DISABLED`：顯示門市名稱即可。  
- `ATTENDED/AUTO`：顯示桌號與本桌摘要（已有 X 單 / Y 未付款）。

---

### 7. POS / 後台
- `DISABLED`：不顯示桌位圖。  
- `ATTENDED`：桌位圖，操作開桌/清桌/轉桌。  
- `AUTO`：桌位圖，操作一鍵清桌/重置 QR，並顯示自動清桌倒數。

---

### 8. 測試策略
- `DISABLED`：併發下單壓測。  
- `ATTENDED`：手動開桌/清桌流程。  
- `AUTO`：自動開桌競態、自動清桌條件測試。

---

## 四、共用 API 建議
- `POST /store/visit/ensure`：統一入口，依 `visit_mode` 決策  
  - `DISABLED` → `{ mode: "DISABLED", sessionId }`  
  - `ATTENDED` → `{ mode: "ATTENDED", visitId? }`  
  - `AUTO` → `{ mode: "AUTO", visitId }`  
- `POST /store/visit/close`：手動清桌，支援 `force`。  
- `POST /store/branch/seat/reset-qrcode`：重置 QR。  
- `GET /store/kds/board`：KDS 讀取，以 `visit_id` 或 `order_id` 聚合。
