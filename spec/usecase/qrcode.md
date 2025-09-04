# 桌位 QRCode 的產生與使用設計

---

## 1) 用到/產生的資料欄位

- `table_info.qr_token`（必用）：**固定桌碼**  
  - 建立桌位或「重置 QR」時，由後端產生（隨機 16 位 base36/hex 皆可）  
  - **不重置就永不變**，印好的 QR 長期有效
- `table_info.id`、`table_info.branch_id`：後端用來把 `qr_token → (branch_id, table_id)` 解析綁定  
- （前端顯示用）`table_info.table_no`, `name`, `pos_x/pos_y/width/height`：不進 QR，只用來渲染與列印標籤  

> **QR 中只放 `qr_token`**，其餘都由後端查表還原。

---

## 2) URL 與 QR 內容

- 建議掃碼 URL（短且穩定）：  
  ```
  https://order.example.com/m/{qr_token}
  ```
- 只放 token，不帶 branchId/tableId 等內部資訊  
- 可選 query（非必要）：`?v=<printed_at_epoch>` 供前端列印版本或防快取使用  

---

## 3) 產生 QR 的時機與流程

### A. 後台「桌位管理」頁面
1. 新增桌位 → 儲存成功，Store 服務為該桌位產生 **一次性** `qr_token` 並落盤  
2. 查看 QR → 前端用 `qr_token` 本地繪製 QR（Canvas/QRCode lib）  
3. 重置 QR → 後端更新 `qr_token`；舊貼紙立即失效，需重印  

### B. 列印建議
- 標籤包含：桌號（粗體）、分店名（小）、URL（小字）、QR Code  
- URL 下可印「桌號＋掃碼點餐提示」  

---

## 4) 客人掃碼後流程

```
客人掃碼 → GET https://order.example.com/m/{qr_token}
  前端呼叫 BFF：POST /public/resolve-seat { token }
  BFF → Store：POST /store/seat/resolve
  Store：
    1. 查表 table_info by qr_token（is_deleted=0, is_active=1）
    2. 取得 branch、table、visit_mode
    3. 依 visit_mode 處理：
       - DISABLED：不追桌 → 建立個人購物車，不創建 visit
       - ATTENDED：需店員開桌 → 無 OPEN visit 時回「請店員帶位」
       - AUTO：若無 OPEN visit → 自動建立 visit(OPEN)
    4. 回傳：branchId, tableId, visitId, bizDate, menuContext 等
```

---

## 5) Resolver API 設計重點

- **BFF 外部 API**：`POST /public/resolve-seat` → `PortalResolveSeatRs`  
- **Feign → Store**：`POST /store/seat/resolve` → `StoreResolveSeatRq/Rs`（放 common/contract）  

### `StoreResolveSeatRs` 建議欄位
- `branchId, tableId, tableNo, visitMode`  
- `visitId`（ATTENDED 模式下可能為 null；AUTO 則會自動建立）  
- `bizDate`（依 branch.biz_day_start_hour 計算）  
- `menuHit`（命中的菜單版本，可選）  
- `tableStatus`（IDLE/IN_USE/OOS）  
- `message/nextWindow`（非營業時段提示）  

### 冪等與併發（AUTO 模式）
- 保證「同桌同時只有一個 OPEN visit」  
- 先查 OPEN，沒有才創建  
- 可用唯一索引 `(table_id, status='OPEN')` 或應用鎖  

---

## 6) 安全性與防護

- **最小暴露**：QR 只含 token  
- **隨機性**：token 長度 ≥16，字符集混合（a-z0-9/base62）  
- **速率限制**：對 `/public/resolve-seat` 做 IP/裝置層 rate limit  
- **撤銷**：  
  - 停用桌位：`is_active = 0` → 掃碼提示「此桌暫停使用」  
  - 重置 QR：更新 token → 舊碼立即失效  

---

## 7) 設計與維運細節

- **印刷前預覽**：前端直接渲染 Canvas，不走後端簽名 URL  
- **短域名（可選）**：`https://csha.me/m/{token}` → 轉址到正式站  
- **AUTO 模式**：第一個掃碼自動開桌，其後同桌共用同一 visitId  
- **ATTENDED 模式**：若尚未開桌，回「等待店員帶位」  
- **DISABLED 模式**：直接用個人購物車，下單不創建 visit  

---

## 8) 例外情境

| 狀況         | 後端行為            | 前端提示                 |
|--------------|---------------------|--------------------------|
| 找不到 token | 404                 | 「此桌位無效或已重置」   |
| 桌位停用     | 200 + 狀態碼        | 「此桌位暫停使用」       |
| 非營業時段   | 200 + `nextWindow` | 「目前非營業，下次時間…」|
| AUTO 撞單    | 保留一個 OPEN visit | 前端不感知，拿同一 visit |
| ATTENDED 未開桌 | `visitId=null`   | 「請先由店員帶位」       |

---

## 9) 端點範例

- **BFF**  
  - `POST /public/resolve-seat` → `PortalResolveSeatRs`  
  - `POST /portal/branchSeat/resetQr` → 重置單桌 QR  

- **BFF → Store**  
  - `POST /store/seat/resolve` → `StoreResolveSeatRq/Rs`  
  - `POST /store/seat/resetQr`  

---

## 10) Resolver 回傳範例

```json
{
  "branchId": 9001,
  "tableId": 581234567890,
  "tableNo": "A1",
  "visitMode": "AUTO",
  "visitId": 81422334455,
  "bizDate": "2025-09-04",
  "tableStatus": "IN_USE",
  "menuHit": { "menuVersionId": 778899, "timeWindow": "08:00-14:00" }
}
```

---

## 總結

**QR 只裝 token，token 永不變（除非重置）。後端以 token 還原桌位與分店，依 `visit_mode` 決定是否自動/人工建立 visit，再命中菜單，進入點餐流程。**
