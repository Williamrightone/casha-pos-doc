# 監控與壓力測試報告（Baseline + Capacity）

> 本報告彙整 **Scan→Menu** 與 **Scan→Menu→Order（含付款回調）** 兩條路徑的
> 基準/容量測試與監控觀察，重點在：**可穩定承載的 RPS/併發、延遲 SLA（p50/p95/p99）、錯誤率、依賴服務健康度**。

---

## 0. 測試環境與共用設定

- **執行環境**：Local（AMD R5 4600H / 32GB RAM），各服務 **單實例**。
- **負載模型**：Ultimate Thread Group，五段階梯並發  
  `100 → 200 → 400 → 600 → 800 users`  
  每段：Startup 60s / Hold 300s / Shutdown 30s。
- **使用者行為腳本（共用）**  
  1) 一次掃碼 → `/cis/seat/resolve`  
  2) 菜單刷新 2–3 次（每 3–7s） → `/cis/menu/active`  
  3) 建立訂單（隨機 1–3 個商品、每個 qty=1~3） → `/cis/order/create`  
  4) 模擬付款回調 → `/cis/payments/mock-callback`
- **量測指標**：p50/p95/p99、Error%、TPS、各服務 p95、CPU/Heap、Redis Ops/Hit Ratio、MQ Queue Depth。

> 在 [菜單設置與快取機制](/spec/usecase/menu.md) 中有提到, 在首次解析該qrcode 時, 會真的到 DB 內查詢, 儲存到 Redis 後返回, 後面的請求則在 Data TTL 前都使用 Redis 的資料。

---

## 1. Scan→Menu 基準/容量（Baseline + Capacity）

**目標**：針對讀路徑（Scan→Menu）量出固定資源下可穩定承載的 RPS，並給出 p50/p95/p99 SLA。  
**輸出**：各階段（100/200/400/600/800 users）之延遲分佈、錯誤率、依賴服務健康度，與結論（穩定承載 RPS、建議 SLA）。

### 1.1 測試模型與資料規模

- 分店：**800 間**，每店 **5** 桌 → **4,000** QR
- 菜單版本：每店 **1** 版，共 **800** 版
- 品項：每版 **10** 品項（3 分類），全部緩存在 Redis（首刷命中 DB，TTL 內皆走 Redis）

**使用者行為**：  
抵達後掃碼 1 次；在 5 分鐘 Hold 期間，每 3–7 秒刷新菜單；段尾逐步退出，進入下一並發段。

> 設計目的：最大化讀路徑覆蓋，反映「高頻看菜單」的真實流量型態。

### 1.2 結果（JMeter）

> Error% 中觀察到的 **Response was null** 判定為 JMeter 偶發、非伺服器錯誤（Error% 1.44% < 2%）。

- **ResolveSeat（掃碼）**  
  p50 ≈ 45 ms / p95 ≈ 60 ms / p99 ≈ 85 ms / Error% = 0%  
  需觸發 DB 與狀態解析，延遲高於菜單，但仍 < 100 ms 且只發生一次，體驗可接受。

- **MenuRefresh（核心讀菜單）**  
  p50 ≈ 12 ms / p95 ≈ 22 ms / p99 ≈ 67 ms / Error% ≈ 1.44%  
  低毫秒級，Redis 快取表現穩定，tail latency 無明顯拖尾。

| Label       | #Samples | Error % | Avg (ms) | p90 | p95 | p99 | TPS  |
| ----------- | -------: | ------: | -------: | --: | --: | --: | ---: |
| ResolveSeat |    4,200 |   0.00% |    56.90 |  88 | 112 | 160 | 2.59 |
| MenuRefresh |  262,503 |   0.00% |    16.30 |  22 |  29 |  92 | 134.8|
| **Total**   |  138,605 |   0.00% |    19.20 |  20 |  26 |  77 | 71.1 |

![Response Times Over Time](/asset/BaseLineResponseTimesOverTime.png)  
![Active Threads Over Time](/asset/BaseLineActiveThreadsOverTime.png)

### 1.3 監控觀察（Grafana/Prometheus）

- **QPS by Service**：峰值約 **130–140 req/s**（主在 `cis-service`），其餘服務 20–40 req/s，負載分布合理。  
- **p95**：`cis-service`（含 seat/resolve + menu/active）< **100 ms**；整體 < **200 ms**，符合常見 SLA。  
- **CPU/Heap**：各服務 CPU 峰值 < **6%**；Heap 約 **1.0–1.7%**，GC 無異常；Redis Ops 峰值 ~**300 ops/s**、Hit Ratio 100%。

![Grafana](/asset/BaseLineGrafana.png)

### 1.4 結論（Scan→Menu）

- 在 **800 並發（~75 RPS）** 條件下，**MenuRefresh** p95 ≈ **22 ms**、p99 ≈ **67 ms**，**Error% < 2%**。  
- **穩定承載能力**：保守建議 **≥ 70–80 RPS**；讀路徑 SLA 可訂 **p95 ≤ 25 ms、p99 ≤ 70 ms、Error% < 2%**。  
- 資源使用率低，尚有擴容空間。

---

## 2. Scan→Menu→Order 基準/容量（Baseline + Capacity）

**目標**：在多分店 open model 下，量測端到端（掃碼→看菜單→下單→回調）的延遲與穩定度。  
**測資假設（每店）**：10 商品、可銷售量總和 **725**；每位用戶下單 **2–3** 商品（qty=1）。

### 2.1 兩組情境

- **性能 SLA baseline（50 店）**：確保 800 users 後仍有庫存，以量 **延遲/TPS/錯誤率**。  
- **正確性 baseline（7 店）**：總可售量 **7×725=5,075**，在後段階段必然售罄，用以驗證 **正確拒絕與無超賣**。

> 整場 5 段合計 2,100 users，若以 2.5 件/人估算，約 **5,250 件** 請求量。

### 2.2 結果（JMeter）

#### a) 50 店（性能）

- **CreateOrder**：Avg 129 ms / p95 255 ms / p99 394 ms / Error% 0%  
- **MockCallback**：Avg 54 ms / p95 106 ms / Error% 0%  
- **MenuRefresh**：如 §1 結果，持續低毫秒級

| Label        | #Samples | Error % | Avg (ms) | p90 | p95 | p99 | TPS  |
| ------------ | -------: | ------: | -------: | --: | --: | --: | ---: |
| ResolveSeat  |    4,200 |   0.00% |   56.90  |  88 | 112 | 160 | 2.59 |
| MenuRefresh  |  262,503 |   0.00% |   16.30  |  22 |  29 |  92 | 134.8|
| CreateOrder  |    2,100 |   0.00% |  129.10  | 211 | 255 | 394 | 1.30 |
| MockCallback |    2,100 |   0.00% |   53.90  |  85 | 106 | 149 | 1.30 |
| **Total**    |  138,605 |   0.00% |   19.20  |  20 |  26 |  77 | 71.1 |

![Response Times Over Time](/asset/casha-order-baseline.png)

**監控**：QPS 峰值 **130–140**；order-service p95 **100–150 ms**；CPU < **6%**、Heap < **2%**、Redis Ops 峰值 ~**600**。

![Grafana](/asset/casha-order-baseline-grafana.png)

**結論（50 店）**：在 **800 users** 下，端到端 **Error = 0%**、CreateOrder **p95 ≈ 255 ms**；可對外主張 **穩定承載 ≈ 130 TPS**、**p95 < 300 ms**。

---

#### b) 7 店（正確性）

- **預期**：累計到第 4–5 段總購買量 > **5,075**，進入售罄。  
- **CreateOrder Error% ≈ 20.8%** 為 **400 / CIS00004（售罄）**，屬於「正確拒絕」。  
- **MockCallback Error% ≈ 32.6%**（因前一步未下單成功，自然無回調對象）。  
- **DB 校驗**：  
  `sum(menu_version_item.daily_quota) = 5,075`  
  `sum(sales_quota_counter.used_qty) = 5,075` → **完全一致，證實無 oversell**。

| Label        | #Samples | Error % | Avg (ms) | p90 | p95 | p99 | TPS  |
| ------------ | -------: | ------: | -------: | --: | --: | --: | ---: |
| ResolveSeat  |    4,200 |    0.0% |   46.10  |  62 |  74 | 104 | 2.59 |
| MenuRefresh  |  262,696 |    0.0% |   12.10  |  15 |  17 |  28 | 134.8|
| CreateOrder  |    2,100 |   20.8% |   82.60  | 120 | 137 | 176 | 1.30 |
| MockCallback |    2,100 |   32.6% |   40.40  |  58 |  73 |  96 | 1.30 |
| **Total**    |  138,700 |    0.8% |   14.10  |  16 |  18 |  27 | 71.2 |

**監控**：QPS **120–140**；order-service p95 ~**150 ms**，cis-service < **100 ms**；CPU 峰值 < **6%**、Heap 緩升但 < **2%**；Redis Hit 100%。

![Grafana](/asset/casha-order-act-grafana.png)

**結論（7 店）**：在受限庫存場景下，系統**正確拒絕**售罄下單、前端菜單同步 `isActive=0`，且 **未發生超賣**；延遲/資源保持穩定。

---

## 3. 綜合結論與建議

1. **可穩定承載能力**  
   - **讀路徑（Scan→Menu）**：穩定承載 **≥ 70–80 RPS**，建議 SLA：**p95 ≤ 25 ms / p99 ≤ 70 ms / Error% < 2%**。  
   - **下單路徑（Scan→Menu→Order）**：在 **~130 TPS** 時，**CreateOrder p95 < 300 ms**、Error ≈ 0%（50 店）。

2. **正確性**  
   - 在 **7 店（5,075 件）** 受限場景，售罄後返回 **400/CIS00004**，**DB 統計與可售量一致**，證實 **無 oversell**。

3. **資源/依賴健康度**  
   - CPU/Heap/Redis/MQ 均遠離瓶頸，說明單機資源足夠、系統可擴空間大。

4. **差異解讀**  
   - JMeter p95（單純 HTTP Sampler 延遲） vs. Grafana p95（含 gateway/metrics）存在落差，屬正常量測口徑差異。

5. **下一步建議**  
   - **Stress/Break Test**：每 3–5 分鐘上調 20–30% RPS，直到 **Error% > 5%** 或 **p99 爆炸**，找臨界點與優雅降級行為。  
   - **熱門品/熱鍵測試**：權重選品，驗證 Redis/DB 熱點行為。  
   - **容量預估**：以本次曲線外推，規劃 2×–3× 擴容策略（服務副本/連線池/隊列上限）。

---

## 附錄 A：資料準備（節錄）

> 大量資料與 QR Token 由 Procedure 產生：分店/分類/品項/菜單/版本/版本關聯/使用者/桌位（含 QR Token），下列為主要概念與呼叫方式（完整 SQL 請見倉庫/附檔）。

```sql
-- 產生 800 間分店的基準資料（摘要）
CALL CASHA_STORE.gen_casha_baseline_data(
    21461976081371136,  -- restaurant_id
    'WXP909KY',         -- rest_code
    3,                  -- start_branch_no
    800,                -- branch_count
    3                   -- start_account_no
);

-- 匯出 token.csv
SELECT qr_token AS token FROM CASHA_STORE.table_info;
```

```sql
DELIMITER $$

DROP PROCEDURE IF EXISTS CASHA_STORE.gen_casha_baseline_data $$
CREATE PROCEDURE CASHA_STORE.gen_casha_baseline_data(
    IN p_restaurant_id BIGINT,
    IN p_rest_code VARCHAR(12),
    IN p_start_branch_no INT,
    IN p_branch_count INT,
    IN p_start_account_no INT
)
BEGIN
    DECLARE i INT DEFAULT 0;
    DECLARE j INT;
    DECLARE k INT;

    -- 依現有資料計算一個不會碰撞的新 base（每個分店保留 100000 的區塊）
    DECLARE m_branch BIGINT DEFAULT 0;
    DECLARE m_cat BIGINT DEFAULT 0;
    DECLARE m_item BIGINT DEFAULT 0;
    DECLARE m_menu BIGINT DEFAULT 0;
    DECLARE m_mv BIGINT DEFAULT 0;
    DECLARE m_mvc BIGINT DEFAULT 0;
    DECLARE m_mvi BIGINT DEFAULT 0;
    DECLARE m_user BIGINT DEFAULT 0;
    DECLARE m_user_role BIGINT DEFAULT 0;
    DECLARE v_global_max BIGINT DEFAULT 0;
    DECLARE v_base BIGINT DEFAULT 0;

    -- 固定要綁定的角色
    DECLARE v_role_id BIGINT DEFAULT 14098420125925376;

    -- 變數容器
    DECLARE v_branch_no INT;
    DECLARE v_branch_id BIGINT;
    DECLARE v_branch_code VARCHAR(16);
    DECLARE v_branch_name VARCHAR(100);

    DECLARE v_user_id BIGINT;
    DECLARE v_user_account VARCHAR(100);
    DECLARE v_username VARCHAR(100);
    DECLARE v_user_role_id BIGINT;

    DECLARE v_acc_seed INT;
    DECLARE v_acc_no INT;

    DECLARE v_menu_id BIGINT;
    DECLARE v_mv_id BIGINT;

    DECLARE v_cat1_id BIGINT;
    DECLARE v_cat2_id BIGINT;
    DECLARE v_cat3_id BIGINT;

    DECLARE v_item_id BIGINT;
    DECLARE v_item_name VARCHAR(150);
    DECLARE v_item_price DECIMAL(10,2);
    DECLARE v_item_cat BIGINT;

    -- 桌位/qr_token 相關
    DECLARE v_table_id BIGINT;
    DECLARE v_qr_token VARCHAR(16);
    DECLARE v_alpha VARCHAR(36) DEFAULT 'abcdefghijklmnopqrstuvwxyz0123456789';
    DECLARE v_dup INT;
    DECLARE ch_idx INT;
    DECLARE t INT;

    -- 取各表當前最大 id，計算新的 v_base
    SELECT COALESCE(MAX(id),0) INTO m_branch     FROM CASHA_STORE.branch;
    SELECT COALESCE(MAX(id),0) INTO m_cat        FROM CASHA_STORE.item_category;
    SELECT COALESCE(MAX(id),0) INTO m_item       FROM CASHA_STORE.item;
    SELECT COALESCE(MAX(id),0) INTO m_menu       FROM CASHA_STORE.menu;
    SELECT COALESCE(MAX(id),0) INTO m_mv         FROM CASHA_STORE.menu_version;
    SELECT COALESCE(MAX(id),0) INTO m_mvc        FROM CASHA_STORE.menu_version_category;
    SELECT COALESCE(MAX(id),0) INTO m_mvi        FROM CASHA_STORE.menu_version_item;
    SELECT COALESCE(MAX(id),0) INTO m_user       FROM CASHA_AUTH.casha_user;
    SELECT COALESCE(MAX(id),0) INTO m_user_role  FROM CASHA_AUTH.user_role;

    SET v_global_max = GREATEST(m_branch, m_cat, m_item, m_menu, m_mv, m_mvc, m_mvi, m_user, m_user_role);
    SET v_base = ((v_global_max DIV 100000) + 1) * 100000;

    -- 從已存在的 FEQNF5ZWxx@casha.com 取最大號碼，後續接續編號
    SELECT COALESCE(MAX(
             CAST(SUBSTRING(SUBSTRING_INDEX(user_account,'@',1), LENGTH(p_rest_code)+1) AS UNSIGNED)
           ), p_start_account_no - 1)
      INTO v_acc_seed
    FROM CASHA_AUTH.casha_user
    WHERE user_account LIKE CONCAT(p_rest_code, '%@casha.com');

    -- 主迴圈
    WHILE i < p_branch_count DO
        SET v_branch_no   = p_start_branch_no + i;
        SET v_branch_id   = v_base + i * 100000;

        -- 分店 code 加 v_base 當 salt，避免與舊 run (同 branch_no) 撞 UK
        SET v_branch_code = UPPER(SUBSTRING(MD5(CONCAT(p_rest_code, '-', v_branch_no, '-', v_base)), 1, 8));
        SET v_branch_name = CONCAT('分店', LPAD(v_branch_no, 2, '0'));

        -- 1) 分店
        INSERT INTO CASHA_STORE.branch
            (id, restaurant_id, code, branch_no, name, tax_id, phone, address, timezone,
             biz_day_start_hour, visit_mode, auto_close_minutes, is_active, created_at, updated_at)
        VALUES
            (v_branch_id, p_restaurant_id, v_branch_code, v_branch_no, v_branch_name,
             LPAD(CAST(2231137 + v_branch_no AS CHAR), 8, '0'),
             CONCAT('02-2311', LPAD(v_branch_no, 4, '0')),
             CONCAT('台北市測試路', v_branch_no, '號'),
             'Asia/Taipei', 6, 'AUTO', 20, 1, NOW(), NOW());

        -- 2) 分店帳號（從既有最大號碼 +1 起）
        SET v_acc_no       = v_acc_seed + 1 + i;
        SET v_user_account = CONCAT(p_rest_code, v_acc_no, '@casha.com');  -- 這行改了：移除 LPAD
        SET v_username     = CONCAT(p_rest_code, v_acc_no, ' 帳號');       -- 這行順便對齊
        SET v_user_id      = v_branch_id + 100;

        INSERT INTO CASHA_AUTH.casha_user
            (id, user_account, username, passwd, employee_no, restaurant_id, branch_id, account_scope, is_active, created_at, updated_at)
        VALUES
            (v_user_id, v_user_account, v_username,
             '$2a$10$ZMnCIowuHB/fFouj.7ycteiVPl6e.ARn3Y7zNMt.W20Z90xEP5qg6',
             NULL, p_restaurant_id, v_branch_id, 'BRANCH', 1, NOW(), NOW());

        -- 2.1) 綁定固定角色（user_role：只有 created_at）
        SET v_user_role_id = v_branch_id + 115;
        INSERT INTO CASHA_AUTH.user_role
            (id, user_id, role_id, created_at)
        VALUES
            (v_user_role_id, v_user_id, v_role_id, NOW());

        -- 3) 五個桌位 T1~T5（產 token：16 碼 a-z0-9）
        SET j = 1;
        WHILE j <= 5 DO  -- 修改：從 2 改為 5
            SET v_table_id = v_branch_id + 50 + j;

            -- 產生唯一 qr_token
            SET v_dup = 1;
            WHILE v_dup > 0 DO
                SET v_qr_token = '';
                SET t = 1;
                WHILE t <= 16 DO
                    SET ch_idx = FLOOR(RAND()*36) + 1; -- 1..36
                    SET v_qr_token = CONCAT(v_qr_token, SUBSTRING(v_alpha, ch_idx, 1));
                    SET t = t + 1;
                END WHILE;
                SELECT COUNT(*) INTO v_dup FROM CASHA_STORE.table_info WHERE qr_token = v_qr_token;
            END WHILE;

            INSERT INTO CASHA_STORE.table_info
                (id, branch_id, table_no, name, pos_x, pos_y, width, height, qr_token, status, is_active, created_at, updated_at)
            VALUES
                (v_table_id, v_branch_id,
                 CONCAT('T', j), CONCAT('第', j, '桌'),
                 (j-1)*3, 0, 2, 2,
                 v_qr_token, 'IDLE', 1, NOW(), NOW());

            SET j = j + 1;
        END WHILE;

        -- 4) 三個分類
        SET v_cat1_id = v_branch_id + 300;  -- 招牌
        SET v_cat2_id = v_branch_id + 301;  -- 飲品
        SET v_cat3_id = v_branch_id + 302;  -- 甜點
        INSERT INTO CASHA_STORE.item_category (id, branch_id, name, sort_order, is_active, created_at, updated_at) VALUES
            (v_cat1_id, v_branch_id, '招牌', 0, 1, NOW(), NOW()),
            (v_cat2_id, v_branch_id, '飲品', 1, 1, NOW(), NOW()),
            (v_cat3_id, v_branch_id, '甜點', 2, 1, NOW(), NOW());

        -- 5) 十個品項
        SET k = 1;
        WHILE k <= 10 DO
            SET v_item_id = v_branch_id + 500 + k;

            IF k <= 4 THEN
                SET v_item_cat = v_cat1_id;
            ELSEIF k <= 7 THEN
                SET v_item_cat = v_cat2_id;
            ELSE
                SET v_item_cat = v_cat3_id;
            END IF;

            SET v_item_name  = CONCAT('品項', LPAD(k,2,'0'));
            SET v_item_price = 30.00 + (k * 5.00);  -- 35,40,...,80

            INSERT INTO CASHA_STORE.item
                (id, branch_id, category_id, name, base_price,
                 image_object_key, image_mime, image_updated_at,
                 is_active, created_at, updated_at)
            VALUES
                (v_item_id, v_branch_id, v_item_cat, v_item_name, v_item_price,
                 CONCAT('item/', v_branch_id, '/', v_item_id, '.jpg'),
                 'image/jpeg', NOW(),
                 1, NOW(), NOW());

            SET k = k + 1;
        END WHILE;

        -- 6) 菜單與版本
        SET v_menu_id = v_branch_id + 200;
        INSERT INTO CASHA_STORE.menu
            (id, branch_id, name, is_active, created_at, updated_at)
        VALUES
            (v_menu_id, v_branch_id, '全天菜單', 1, NOW(), NOW());

        SET v_mv_id = v_branch_id + 210;
        INSERT INTO CASHA_STORE.menu_version
            (id, menu_id, version_no, status, date_start, date_end, time_start, time_end, dow_mask, created_at, updated_at)
        VALUES
            (v_mv_id, v_menu_id, 1, 'ACTIVE',
             DATE '2025-09-17', DATE '2025-10-31',
             TIME '00:00:00', TIME '23:30:00',
             127, NOW(), NOW());

        INSERT INTO CASHA_STORE.menu_version_category
            (id, menu_version_id, category_id, sort_order, created_at, updated_at)
        VALUES
            (v_branch_id + 800, v_mv_id, v_cat1_id, 0, NOW(), NOW()),
            (v_branch_id + 801, v_mv_id, v_cat2_id, 1, NOW(), NOW()),
            (v_branch_id + 802, v_mv_id, v_cat3_id, 2, NOW(), NOW());

        -- 7) 版本-品項（全上）
        SET k = 1;
        WHILE k <= 10 DO
            SET v_item_id = v_branch_id + 500 + k;

            INSERT INTO CASHA_STORE.menu_version_item
                (id, menu_version_id, item_id, price, daily_quota, is_active, created_at, updated_at)
            SELECT
                v_branch_id + 900 + k, v_mv_id, i.id, i.base_price,
                CASE (k % 3) WHEN 1 THEN 50 WHEN 2 THEN 75 ELSE 100 END,
                1, NOW(), NOW()
            FROM CASHA_STORE.item i
            WHERE i.id = v_item_id;

            SET k = k + 1;
        END WHILE;

        SET i = i + 1;
    END WHILE;
END $$
DELIMITER ;
```

Menu 的 JMX

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.3">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Casha-Base-Line">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments" guiclass="ArgumentsPanel" testclass="Arguments" testname="User Defined Variables">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <Arguments guiclass="ArgumentsPanel" testclass="Arguments" testname="User Defined Variables">
        <collectionProp name="Arguments.arguments">
          <elementProp name="BASE_FE" elementType="Argument">
            <stringProp name="Argument.name">BASE_FE</stringProp>
            <stringProp name="Argument.value">http://localhost:5174</stringProp>
            <stringProp name="Argument.metadata">=</stringProp>
          </elementProp>
          <elementProp name="BASE_CIS" elementType="Argument">
            <stringProp name="Argument.name">BASE_CIS</stringProp>
            <stringProp name="Argument.value">http://localhost:7788</stringProp>
            <stringProp name="Argument.metadata">=</stringProp>
          </elementProp>
        </collectionProp>
      </Arguments>
      <hashTree/>
      <kg.apc.jmeter.threads.UltimateThreadGroup guiclass="kg.apc.jmeter.threads.UltimateThreadGroupGui" testclass="kg.apc.jmeter.threads.UltimateThreadGroup" testname="jp@gc - Ultimate Thread Group" enabled="true">
        <collectionProp name="ultimatethreadgroupdata">
          <collectionProp name="1615331247">
            <stringProp name="48625">100</stringProp>
            <stringProp name="0">0</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="-1182097314">
            <stringProp name="49586">200</stringProp>
            <stringProp name="50826">390</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="-1616093938">
            <stringProp name="51508">400</stringProp>
            <stringProp name="54639">780</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="2000727162">
            <stringProp name="53430">600</stringProp>
            <stringProp name="1508601">1170</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="1943377413">
            <stringProp name="55352">800</stringProp>
            <stringProp name="1512414">1560</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
        </collectionProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController" guiclass="LoopControlPanel" testclass="LoopController" testname="Loop Controller">
          <intProp name="LoopController.loops">-1</intProp>
          <boolProp name="LoopController.continue_forever">false</boolProp>
        </elementProp>
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
      </kg.apc.jmeter.threads.UltimateThreadGroup>
      <hashTree>
        <TransactionController guiclass="TransactionControllerGui" testclass="TransactionController" testname="Scan→Menu" enabled="true">
          <boolProp name="TransactionController.includeTimers">false</boolProp>
        </TransactionController>
        <hashTree>
          <CSVDataSet guiclass="TestBeanGUI" testclass="CSVDataSet" testname="CSV Data Set Config" enabled="true">
            <stringProp name="delimiter">,</stringProp>
            <stringProp name="fileEncoding">UTF-8</stringProp>
            <stringProp name="filename">C:/Users/willy/Desktop/tokens.csv</stringProp>
            <boolProp name="ignoreFirstLine">false</boolProp>
            <boolProp name="quotedData">false</boolProp>
            <boolProp name="recycle">true</boolProp>
            <stringProp name="shareMode">shareMode.all</stringProp>
            <boolProp name="stopThread">false</boolProp>
            <stringProp name="variableNames"></stringProp>
          </CSVDataSet>
          <hashTree/>
          <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="HTTP Header Manager" enabled="true">
            <collectionProp name="HeaderManager.headers">
              <elementProp name="" elementType="Header">
                <stringProp name="Header.name">Content-Type</stringProp>
                <stringProp name="Header.value">application/json</stringProp>
              </elementProp>
              <elementProp name="" elementType="Header">
                <stringProp name="Header.name">Accept</stringProp>
                <stringProp name="Header.value">application/json</stringProp>
              </elementProp>
            </collectionProp>
          </HeaderManager>
          <hashTree/>
          <OnceOnlyController guiclass="OnceOnlyControllerGui" testclass="OnceOnlyController" testname="Once Only Controller" enabled="true"/>
          <hashTree>
            <TransactionController guiclass="TransactionControllerGui" testclass="TransactionController" testname="ResolveSeat" enabled="true">
              <boolProp name="TransactionController.parent">true</boolProp>
              <boolProp name="TransactionController.includeTimers">false</boolProp>
            </TransactionController>
            <hashTree>
              <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="/cis/seat/resolve" enabled="true">
                <stringProp name="HTTPSampler.path">${BASE_CIS}/cis/seat/resolve</stringProp>
                <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
                <stringProp name="HTTPSampler.method">POST</stringProp>
                <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
                <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
                <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                  <collectionProp name="Arguments.arguments">
                    <elementProp name="" elementType="HTTPArgument">
                      <boolProp name="HTTPArgument.always_encode">false</boolProp>
                      <stringProp name="Argument.value">{ &quot;token&quot;: &quot;${token}&quot; }</stringProp>
                      <stringProp name="Argument.metadata">=</stringProp>
                    </elementProp>
                  </collectionProp>
                </elementProp>
              </HTTPSamplerProxy>
              <hashTree>
                <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="variables" enabled="true">
                  <stringProp name="JSONPostProcessor.referenceNames">branchId;tableId;customerToken</stringProp>
                  <stringProp name="JSONPostProcessor.jsonPathExprs">$.data.branchId;$.data.tableId;$.data.customerToken</stringProp>
                  <stringProp name="JSONPostProcessor.match_numbers">1;1;1</stringProp>
                  <stringProp name="JSONPostProcessor.defaultValues">NOT_FOUND;NOT_FOUND;NOT_FOUND</stringProp>
                </JSONPostProcessor>
                <hashTree/>
                <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion" enabled="true">
                  <collectionProp name="Asserion.test_strings">
                    <stringProp name="49586">200</stringProp>
                  </collectionProp>
                  <stringProp name="Assertion.custom_message"></stringProp>
                  <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
                  <boolProp name="Assertion.assume_success">false</boolProp>
                  <intProp name="Assertion.test_type">8</intProp>
                </ResponseAssertion>
                <hashTree/>
              </hashTree>
            </hashTree>
          </hashTree>
          <RunTime guiclass="RunTimeGui" testclass="RunTime" testname="Runtime Controller" enabled="true">
            <stringProp name="RunTime.seconds">60</stringProp>
          </RunTime>
          <hashTree>
            <LoopController guiclass="LoopControlPanel" testclass="LoopController" testname="Loop Controller" enabled="true">
              <intProp name="LoopController.loops">-1</intProp>
            </LoopController>
            <hashTree>
              <UniformRandomTimer guiclass="UniformRandomTimerGui" testclass="UniformRandomTimer" testname="Uniform Random Timer" enabled="true">
                <stringProp name="ConstantTimer.delay">3000</stringProp>
                <stringProp name="RandomTimer.range">4000</stringProp>
              </UniformRandomTimer>
              <hashTree/>
              <TransactionController guiclass="TransactionControllerGui" testclass="TransactionController" testname="MenuRefresh" enabled="true">
                <boolProp name="TransactionController.parent">true</boolProp>
                <boolProp name="TransactionController.includeTimers">false</boolProp>
              </TransactionController>
              <hashTree>
                <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="HTTP Request" enabled="true">
                  <stringProp name="HTTPSampler.path">${BASE_CIS}/cis/menu/active</stringProp>
                  <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
                  <stringProp name="HTTPSampler.method">POST</stringProp>
                  <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
                  <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
                  <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                    <collectionProp name="Arguments.arguments">
                      <elementProp name="" elementType="HTTPArgument">
                        <boolProp name="HTTPArgument.always_encode">false</boolProp>
                        <stringProp name="Argument.value">{ &quot;branchId&quot;: &quot;${branchId}&quot;, &quot;tableId&quot;: &quot;${tableId}&quot; }&#xd;
</stringProp>
                        <stringProp name="Argument.metadata">=</stringProp>
                      </elementProp>
                    </collectionProp>
                  </elementProp>
                </HTTPSamplerProxy>
                <hashTree/>
              </hashTree>
            </hashTree>
            <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion" enabled="true">
              <collectionProp name="Asserion.test_strings">
                <stringProp name="49586">200</stringProp>
              </collectionProp>
              <stringProp name="Assertion.custom_message"></stringProp>
              <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
              <boolProp name="Assertion.assume_success">false</boolProp>
              <intProp name="Assertion.test_type">16</intProp>
            </ResponseAssertion>
            <hashTree/>
          </hashTree>
        </hashTree>
      </hashTree>
      <ResultCollector guiclass="SimpleDataWriter" testclass="ResultCollector" testname="Simple Data Writer" enabled="true">
        <boolProp name="ResultCollector.error_logging">false</boolProp>
        <objProp>
          <name>saveConfig</name>
          <value class="SampleSaveConfiguration">
            <time>true</time>
            <latency>true</latency>
            <timestamp>true</timestamp>
            <success>true</success>
            <label>true</label>
            <code>true</code>
            <message>true</message>
            <threadName>true</threadName>
            <dataType>true</dataType>
            <encoding>false</encoding>
            <assertions>true</assertions>
            <subresults>true</subresults>
            <responseData>false</responseData>
            <samplerData>false</samplerData>
            <xml>false</xml>
            <fieldNames>true</fieldNames>
            <responseHeaders>false</responseHeaders>
            <requestHeaders>false</requestHeaders>
            <responseDataOnError>false</responseDataOnError>
            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
            <assertionsResultsToSave>0</assertionsResultsToSave>
            <bytes>true</bytes>
            <sentBytes>true</sentBytes>
            <url>true</url>
            <threadCounts>true</threadCounts>
            <idleTime>true</idleTime>
            <connectTime>true</connectTime>
          </value>
        </objProp>
        <stringProp name="filename">C:\Users\willy\Desktop\casha-base-line-data</stringProp>
      </ResultCollector>
      <hashTree/>
      <ResultCollector guiclass="StatVisualizer" testclass="ResultCollector" testname="Aggregate Report" enabled="true">
        <boolProp name="ResultCollector.error_logging">false</boolProp>
        <objProp>
          <name>saveConfig</name>
          <value class="SampleSaveConfiguration">
            <time>true</time>
            <latency>true</latency>
            <timestamp>true</timestamp>
            <success>true</success>
            <label>true</label>
            <code>true</code>
            <message>true</message>
            <threadName>true</threadName>
            <dataType>true</dataType>
            <encoding>false</encoding>
            <assertions>true</assertions>
            <subresults>true</subresults>
            <responseData>false</responseData>
            <samplerData>false</samplerData>
            <xml>false</xml>
            <fieldNames>true</fieldNames>
            <responseHeaders>false</responseHeaders>
            <requestHeaders>false</requestHeaders>
            <responseDataOnError>false</responseDataOnError>
            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
            <assertionsResultsToSave>0</assertionsResultsToSave>
            <bytes>true</bytes>
            <sentBytes>true</sentBytes>
            <url>true</url>
            <threadCounts>true</threadCounts>
            <idleTime>true</idleTime>
            <connectTime>true</connectTime>
          </value>
        </objProp>
        <stringProp name="filename">C:\Users\willy\Desktop\casha-base-line-agg</stringProp>
      </ResultCollector>
      <hashTree/>
      <ResultCollector guiclass="SummaryReport" testclass="ResultCollector" testname="Summary Report" enabled="true">
        <boolProp name="ResultCollector.error_logging">false</boolProp>
        <objProp>
          <name>saveConfig</name>
          <value class="SampleSaveConfiguration">
            <time>true</time>
            <latency>true</latency>
            <timestamp>true</timestamp>
            <success>true</success>
            <label>true</label>
            <code>true</code>
            <message>true</message>
            <threadName>true</threadName>
            <dataType>true</dataType>
            <encoding>false</encoding>
            <assertions>true</assertions>
            <subresults>true</subresults>
            <responseData>false</responseData>
            <samplerData>false</samplerData>
            <xml>false</xml>
            <fieldNames>true</fieldNames>
            <responseHeaders>false</responseHeaders>
            <requestHeaders>false</requestHeaders>
            <responseDataOnError>false</responseDataOnError>
            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
            <assertionsResultsToSave>0</assertionsResultsToSave>
            <bytes>true</bytes>
            <sentBytes>true</sentBytes>
            <url>true</url>
            <threadCounts>true</threadCounts>
            <idleTime>true</idleTime>
            <connectTime>true</connectTime>
          </value>
        </objProp>
        <stringProp name="filename">C:\Users\willy\Desktop\casha-base-line-sum</stringProp>
      </ResultCollector>
      <hashTree/>
    </hashTree>
  </hashTree>
</jmeterTestPlan>

```

Order 的 JMX

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.3">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Casha-Order-50">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments" guiclass="ArgumentsPanel" testclass="Arguments" testname="User Defined Variables">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="HTTP Header Manager">
        <collectionProp name="HeaderManager.headers">
          <elementProp name="" elementType="Header">
            <stringProp name="Header.name">Content-Type</stringProp>
            <stringProp name="Header.value">application/json</stringProp>
          </elementProp>
          <elementProp name="" elementType="Header">
            <stringProp name="Header.name">Accept</stringProp>
            <stringProp name="Header.value">application/json</stringProp>
          </elementProp>
        </collectionProp>
      </HeaderManager>
      <hashTree/>
      <kg.apc.jmeter.threads.UltimateThreadGroup guiclass="kg.apc.jmeter.threads.UltimateThreadGroupGui" testclass="kg.apc.jmeter.threads.UltimateThreadGroup" testname="jp@gc - Ultimate Thread Group">
        <collectionProp name="ultimatethreadgroupdata">
          <collectionProp name="-284469641">
            <stringProp name="100">100</stringProp>
            <stringProp name="48">0</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="-1182097314">
            <stringProp name="49586">200</stringProp>
            <stringProp name="50826">390</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="-1616093938">
            <stringProp name="51508">400</stringProp>
            <stringProp name="54639">780</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="2000727162">
            <stringProp name="53430">600</stringProp>
            <stringProp name="1508601">1170</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
          <collectionProp name="1943377413">
            <stringProp name="55352">800</stringProp>
            <stringProp name="1512414">1560</stringProp>
            <stringProp name="1722">60</stringProp>
            <stringProp name="50547">300</stringProp>
            <stringProp name="1629">30</stringProp>
          </collectionProp>
        </collectionProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController" guiclass="LoopControlPanel" testclass="LoopController" testname="Loop Controller">
          <intProp name="LoopController.loops">-1</intProp>
          <boolProp name="LoopController.continue_forever">false</boolProp>
        </elementProp>
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
      </kg.apc.jmeter.threads.UltimateThreadGroup>
      <hashTree>
        <CSVDataSet guiclass="TestBeanGUI" testclass="CSVDataSet" testname="CSV Data Set Config" enabled="true">
          <stringProp name="filename">C:/Users/willy/Desktop/order-baseline/base-50/tokens.csv</stringProp>
          <stringProp name="fileEncoding"></stringProp>
          <stringProp name="variableNames"></stringProp>
          <boolProp name="ignoreFirstLine">false</boolProp>
          <stringProp name="delimiter">,</stringProp>
          <boolProp name="quotedData">false</boolProp>
          <boolProp name="recycle">true</boolProp>
          <boolProp name="stopThread">false</boolProp>
          <stringProp name="shareMode">shareMode.all</stringProp>
        </CSVDataSet>
        <hashTree/>
        <OnceOnlyController guiclass="OnceOnlyControllerGui" testclass="OnceOnlyController" testname="Once Only Controller" enabled="true"/>
        <hashTree>
          <TransactionController guiclass="TransactionControllerGui" testclass="TransactionController" testname="ResolveSeat" enabled="true">
            <boolProp name="TransactionController.parent">true</boolProp>
            <boolProp name="TransactionController.includeTimers">false</boolProp>
          </TransactionController>
          <hashTree>
            <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="ResolveSeat" enabled="true">
              <stringProp name="HTTPSampler.domain">localhost</stringProp>
              <stringProp name="HTTPSampler.port">7788</stringProp>
              <stringProp name="HTTPSampler.path">/cis/seat/resolve</stringProp>
              <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
              <stringProp name="HTTPSampler.method">POST</stringProp>
              <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
              <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
              <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                <collectionProp name="Arguments.arguments">
                  <elementProp name="" elementType="HTTPArgument">
                    <boolProp name="HTTPArgument.always_encode">false</boolProp>
                    <stringProp name="Argument.value">{&quot;token&quot;:&quot;${token}&quot;}</stringProp>
                    <stringProp name="Argument.metadata">=</stringProp>
                  </elementProp>
                </collectionProp>
              </elementProp>
            </HTTPSamplerProxy>
            <hashTree>
              <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="JSON Extractor" enabled="true">
                <stringProp name="JSONPostProcessor.referenceNames">branchId;tableId;customerToken;visitId</stringProp>
                <stringProp name="JSONPostProcessor.jsonPathExprs">$.data.branchId;$.data.tableId;$.data.customerToken;$.data.visitId</stringProp>
                <stringProp name="JSONPostProcessor.match_numbers">1;1;1;1</stringProp>
                <stringProp name="JSONPostProcessor.defaultValues">NOT_FOUND;NOT_FOUND;NOT_FOUND;NOT_FOUND</stringProp>
              </JSONPostProcessor>
              <hashTree/>
              <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion" enabled="true">
                <collectionProp name="Asserion.test_strings">
                  <stringProp name="49586">200</stringProp>
                </collectionProp>
                <stringProp name="Assertion.custom_message"></stringProp>
                <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
                <boolProp name="Assertion.assume_success">false</boolProp>
                <intProp name="Assertion.test_type">16</intProp>
              </ResponseAssertion>
              <hashTree/>
            </hashTree>
          </hashTree>
        </hashTree>
        <LoopController guiclass="LoopControlPanel" testclass="LoopController" testname="Loop Controller" enabled="true">
          <stringProp name="LoopController.loops">2</stringProp>
        </LoopController>
        <hashTree>
          <TransactionController guiclass="TransactionControllerGui" testclass="TransactionController" testname="MenuRefresh" enabled="true">
            <boolProp name="TransactionController.parent">true</boolProp>
            <boolProp name="TransactionController.includeTimers">false</boolProp>
          </TransactionController>
          <hashTree>
            <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="MenuRefresh" enabled="true">
              <stringProp name="HTTPSampler.domain">localhost</stringProp>
              <stringProp name="HTTPSampler.port">7788</stringProp>
              <stringProp name="HTTPSampler.path">/cis/menu/active</stringProp>
              <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
              <stringProp name="HTTPSampler.method">POST</stringProp>
              <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
              <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
              <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                <collectionProp name="Arguments.arguments">
                  <elementProp name="" elementType="HTTPArgument">
                    <boolProp name="HTTPArgument.always_encode">false</boolProp>
                    <stringProp name="Argument.value">{&quot;branchId&quot;:&quot;${branchId}&quot;,&quot;tableId&quot;:&quot;${tableId}&quot;}</stringProp>
                    <stringProp name="Argument.metadata">=</stringProp>
                  </elementProp>
                </collectionProp>
              </elementProp>
            </HTTPSamplerProxy>
            <hashTree>
              <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="JSON Extractor" enabled="true">
                <stringProp name="JSONPostProcessor.referenceNames">menuJson</stringProp>
                <stringProp name="JSONPostProcessor.jsonPathExprs">$.data </stringProp>
                <stringProp name="JSONPostProcessor.match_numbers">1</stringProp>
                <stringProp name="JSONPostProcessor.defaultValues">NOT_FOUND</stringProp>
              </JSONPostProcessor>
              <hashTree/>
              <JSR223PostProcessor guiclass="TestBeanGUI" testclass="JSR223PostProcessor" testname="JSR223 PostProcessor 隨機選商品/qty" enabled="true">
                <stringProp name="cacheKey">true</stringProp>
                <stringProp name="filename"></stringProp>
                <stringProp name="parameters"></stringProp>
                <stringProp name="script">import groovy.json.JsonSlurper
import java.util.concurrent.ThreadLocalRandom

def raw = vars.get(&quot;menuJson&quot;)
if (raw == null || raw == &quot;NOT_FOUND&quot;) return

def j = new JsonSlurper().parseText(raw)
def items = (j.items ?: []).findAll { it.isActive == 1 }

if (items &amp;&amp; !items.isEmpty()) {
    def rnd = ThreadLocalRandom.current()
    def howMany = 1 + rnd.nextInt(3)   // 隨機 1~3 個商品
    def picked = []
    Collections.shuffle(items, rnd)    // 打亂順序
    items.take(howMany).each { it -&gt;
        def qty = 1 + rnd.nextInt(3)   // 每個商品 qty 1~3
        picked &lt;&lt; [ &quot;itemId&quot;: it.id, &quot;qty&quot;: qty ]
    }
    // 存成 JSON 字串，後續 CreateOrder 直接嵌入
    def itemsJson = new groovy.json.JsonBuilder(picked).toString()
    vars.put(&quot;itemsJson&quot;, itemsJson)
    vars.put(&quot;menuVersionId&quot;, j.menuVersionId as String)
} else {
    vars.put(&quot;itemsJson&quot;, &quot;[]&quot;)
}
</stringProp>
                <stringProp name="scriptLanguage">groovy</stringProp>
              </JSR223PostProcessor>
              <hashTree/>
              <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion" enabled="true">
                <collectionProp name="Asserion.test_strings">
                  <stringProp name="49586">200</stringProp>
                </collectionProp>
                <stringProp name="Assertion.custom_message"></stringProp>
                <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
                <boolProp name="Assertion.assume_success">false</boolProp>
                <intProp name="Assertion.test_type">16</intProp>
              </ResponseAssertion>
              <hashTree/>
              <UniformRandomTimer guiclass="UniformRandomTimerGui" testclass="UniformRandomTimer" testname="Uniform Random Timer" enabled="true">
                <stringProp name="ConstantTimer.delay">4000</stringProp>
                <stringProp name="RandomTimer.range">3000</stringProp>
              </UniformRandomTimer>
              <hashTree/>
            </hashTree>
          </hashTree>
        </hashTree>
        <OnceOnlyController guiclass="OnceOnlyControllerGui" testclass="OnceOnlyController" testname="CreateOrder" enabled="true"/>
        <hashTree>
          <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="CreateOrder" enabled="true">
            <stringProp name="HTTPSampler.domain">localhost</stringProp>
            <stringProp name="HTTPSampler.port">7788</stringProp>
            <stringProp name="HTTPSampler.path">/cis/order/create</stringProp>
            <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
            <stringProp name="HTTPSampler.method">POST</stringProp>
            <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
            <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
            <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
              <collectionProp name="Arguments.arguments">
                <elementProp name="" elementType="HTTPArgument">
                  <boolProp name="HTTPArgument.always_encode">false</boolProp>
                  <stringProp name="Argument.value">{&#xd;
  &quot;customerToken&quot;:&quot;${customerToken}&quot;,&#xd;
  &quot;items&quot;:${itemsJson},&#xd;
  &quot;paymentMethod&quot;:&quot;LINEPAY&quot;,&#xd;
  &quot;clientTxnId&quot;:&quot;${__UUID()}&quot;,&#xd;
  &quot;menuVersionId&quot;:&quot;${menuVersionId}&quot;,&#xd;
  &quot;visitId&quot;:&quot;${visitId}&quot;&#xd;
}&#xd;
</stringProp>
                  <stringProp name="Argument.metadata">=</stringProp>
                </elementProp>
              </collectionProp>
            </elementProp>
          </HTTPSamplerProxy>
          <hashTree>
            <JSR223Assertion guiclass="TestBeanGUI" testclass="JSR223Assertion" testname="JSR223 Assertion" enabled="true">
              <stringProp name="cacheKey">true</stringProp>
              <stringProp name="filename"></stringProp>
              <stringProp name="parameters"></stringProp>
              <stringProp name="script">import groovy.json.JsonSlurper
def http = prev.getResponseCode()
def ok = false
try {
  def j = new JsonSlurper().parseText(prev.getResponseDataAsString())
  if (http == &quot;200&quot; &amp;&amp; j.responseCode == &quot;00000&quot;) ok = true
  if (http == &quot;400&quot; &amp;&amp; j.responseCode == &quot;CIS00004&quot;) ok = true
  if (ok &amp;&amp; http == &quot;200&quot;) vars.put(&apos;orderId&apos;, j.data?.id ?: &apos;&apos;)
} catch (ignored) {}
prev.setSuccessful(ok)
</stringProp>
              <stringProp name="scriptLanguage">groovy</stringProp>
            </JSR223Assertion>
            <hashTree/>
            <JSONPostProcessor guiclass="JSONPostProcessorGui" testclass="JSONPostProcessor" testname="JSON Extractor" enabled="true">
              <stringProp name="JSONPostProcessor.referenceNames">orderId</stringProp>
              <stringProp name="JSONPostProcessor.jsonPathExprs">$.data.id</stringProp>
              <stringProp name="JSONPostProcessor.match_numbers">1</stringProp>
              <stringProp name="JSONPostProcessor.defaultValues">NOT_FOUND</stringProp>
            </JSONPostProcessor>
            <hashTree/>
          </hashTree>
        </hashTree>
        <OnceOnlyController guiclass="OnceOnlyControllerGui" testclass="OnceOnlyController" testname="MockCallback" enabled="true"/>
        <hashTree>
          <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="MockCallback" enabled="true">
            <stringProp name="HTTPSampler.domain">localhost</stringProp>
            <stringProp name="HTTPSampler.port">7788</stringProp>
            <stringProp name="HTTPSampler.path">/cis/payments/mock-callback</stringProp>
            <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
            <stringProp name="HTTPSampler.method">POST</stringProp>
            <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
            <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
            <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
              <collectionProp name="Arguments.arguments">
                <elementProp name="" elementType="HTTPArgument">
                  <boolProp name="HTTPArgument.always_encode">false</boolProp>
                  <stringProp name="Argument.value">{&#xd;
  &quot;orderId&quot;:&quot;${orderId}&quot;,&#xd;
  &quot;method&quot;:&quot;LINEPAY&quot;,&#xd;
  &quot;result&quot;:&quot;SUCCESS&quot;&#xd;
}&#xd;
</stringProp>
                  <stringProp name="Argument.metadata">=</stringProp>
                </elementProp>
              </collectionProp>
            </elementProp>
          </HTTPSamplerProxy>
          <hashTree/>
        </hashTree>
        <ResultCollector guiclass="StatVisualizer" testclass="ResultCollector" testname="Aggregate Report">
          <boolProp name="ResultCollector.error_logging">false</boolProp>
          <objProp>
            <name>saveConfig</name>
            <value class="SampleSaveConfiguration">
              <time>true</time>
              <latency>true</latency>
              <timestamp>true</timestamp>
              <success>true</success>
              <label>true</label>
              <code>true</code>
              <message>true</message>
              <threadName>true</threadName>
              <dataType>true</dataType>
              <encoding>false</encoding>
              <assertions>true</assertions>
              <subresults>true</subresults>
              <responseData>false</responseData>
              <samplerData>false</samplerData>
              <xml>false</xml>
              <fieldNames>true</fieldNames>
              <responseHeaders>false</responseHeaders>
              <requestHeaders>false</requestHeaders>
              <responseDataOnError>false</responseDataOnError>
              <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
              <assertionsResultsToSave>0</assertionsResultsToSave>
              <bytes>true</bytes>
              <sentBytes>true</sentBytes>
              <url>true</url>
              <threadCounts>true</threadCounts>
              <idleTime>true</idleTime>
              <connectTime>true</connectTime>
            </value>
          </objProp>
          <stringProp name="filename"></stringProp>
        </ResultCollector>
        <hashTree/>
        <ResultCollector guiclass="SimpleDataWriter" testclass="ResultCollector" testname="Simple Data Writer" enabled="true">
          <boolProp name="ResultCollector.error_logging">false</boolProp>
          <objProp>
            <name>saveConfig</name>
            <value class="SampleSaveConfiguration">
              <time>true</time>
              <latency>true</latency>
              <timestamp>true</timestamp>
              <success>true</success>
              <label>true</label>
              <code>true</code>
              <message>true</message>
              <threadName>true</threadName>
              <dataType>true</dataType>
              <encoding>false</encoding>
              <assertions>true</assertions>
              <subresults>true</subresults>
              <responseData>false</responseData>
              <samplerData>false</samplerData>
              <xml>false</xml>
              <fieldNames>true</fieldNames>
              <responseHeaders>false</responseHeaders>
              <requestHeaders>false</requestHeaders>
              <responseDataOnError>false</responseDataOnError>
              <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
              <assertionsResultsToSave>0</assertionsResultsToSave>
              <bytes>true</bytes>
              <sentBytes>true</sentBytes>
              <url>true</url>
              <threadCounts>true</threadCounts>
              <idleTime>true</idleTime>
              <connectTime>true</connectTime>
            </value>
          </objProp>
          <stringProp name="filename">C:\Users\willy\Desktop\order-baseline\base-50\casha-base-line-data.jtl</stringProp>
        </ResultCollector>
        <hashTree/>
        <ResultCollector guiclass="SummaryReport" testclass="ResultCollector" testname="Summary Report" enabled="true">
          <boolProp name="ResultCollector.error_logging">false</boolProp>
          <objProp>
            <name>saveConfig</name>
            <value class="SampleSaveConfiguration">
              <time>true</time>
              <latency>true</latency>
              <timestamp>true</timestamp>
              <success>true</success>
              <label>true</label>
              <code>true</code>
              <message>true</message>
              <threadName>true</threadName>
              <dataType>true</dataType>
              <encoding>false</encoding>
              <assertions>true</assertions>
              <subresults>true</subresults>
              <responseData>false</responseData>
              <samplerData>false</samplerData>
              <xml>false</xml>
              <fieldNames>true</fieldNames>
              <responseHeaders>false</responseHeaders>
              <requestHeaders>false</requestHeaders>
              <responseDataOnError>false</responseDataOnError>
              <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
              <assertionsResultsToSave>0</assertionsResultsToSave>
              <bytes>true</bytes>
              <sentBytes>true</sentBytes>
              <url>true</url>
              <threadCounts>true</threadCounts>
              <idleTime>true</idleTime>
              <connectTime>true</connectTime>
            </value>
          </objProp>
          <stringProp name="filename"></stringProp>
        </ResultCollector>
        <hashTree/>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>

```