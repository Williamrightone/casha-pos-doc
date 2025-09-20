# 監控與壓力測試

關於壓力測試, 會注重於在特定情境之下, 系統能夠運行的極限。其中比較費工的點分為事前的準備, 以及 TestCase 的定義, 這些內容都需要在執行之前有完善的計畫。

## 1. 容量/基準（Baseline + Capacity）測試

> 目標：在多分店 open model 下，Scan→Menu（讀路徑)，量出系統在固定資源上的可穩定 RPS，並給出 p50/p95/p99 SLA。

> 輸出：每一階段（100/200/400/600/800 RPS）的 p50/p95/p99、錯誤率、依賴服務指標；結論（穩定承載 RPS 與建議 SLA）。

測試環境是我的 Local 電腦, AMD R5 4600H, 32GB Ram, 開啟全服務單 instance。

### Test Case

用戶掃碼進入 web app 時, 會先將 qrcode 解析成 branchId 和 TableId 返回, 前端轉導後再拿他們去查詢菜單。

因此會執行:

* ResolveSeat 一次
* MenuRefresh 多次

> 而在 [菜單設置與快取機制](/spec/usecase/menu.md) 中有提到, 在首次解析該qrcode 時, 會真的到 DB 內查詢, 儲存到 Redis 後返回, 後面的請求則在 Data TTL 前都使用 Redis 的資料。

因此這次的 BaseLine 設計:

* 800 家分店 (800 × 1)
* 4,000 張座位(等同 4000 個 QR Code, 每家分店 5 張桌位 = 800 × 5)
* 800個菜單版本 (每家 1 版本 = 800 × 1)
* 每個版本 3 個商品分類
* 每個版本 10 個商品

模擬機制為:

> 起始 100 位用戶, 在 1 分鐘內全部抵達, 抵達後會執行掃描 Qrcode 1 次, 並在 5 分鐘內, 每隔 3 ~ 7 秒會發查一次菜單請求, 五分鐘後客戶退場, 在後續的階段改為 200, 400, 600, 800 位用戶。

![Expect Base Line](/asset/casha-expect-baseline.png)

### a. 測資準備 Procedure

產生

* 800家分店 (800 × 1)
* 2,400個分類 (800 × 3) - 每家3分類
* 8,000個品項 (800 × 10) - 每家10品項
* 800個菜單 (800 × 1) - 每家1菜單
* 800個菜單版本 (800 × 1) - 每家1版本
* 2,400個版本-分類關聯 (800 × 3) - 每個版本關聯3個分類
* 8,000個版本-品項關聯 (800 × 10) - 每個版本關聯10個品項
* 800個使用者帳號 (800 × 1) - 每家1帳號
* 4,000張座位 (800 × 5) - 每家5張桌位

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

下一步是把建立出來的餐廳帳號與 Id 放入, 便可執行 procedure。

```sql
CALL CASHA_STORE.gen_casha_baseline_data(
    21461976081371136,  -- restaurant_id
    'WXP909KY',         -- rest_code
    3,                  -- start_branch_no (從3號開始)
    800,                -- branch_count (800間分店)
    3                   -- start_account_no
);
```

### b. 匯出 token.csv

```sql
select qr_token as token from CASHA_STORE.table_info
```

## Test Result

### Statistic

| Label       | #Samples | FAIL | Error % | Average | Min | Max  | Median | 90th pct | 95th pct | 99th pct | Throughput (t/s) | Received (KB/s) | Sent (KB/s) |
|-------------|----------|------|---------|---------|-----|------|--------|----------|----------|----------|------------------|-----------------|-------------|
| MenuRefresh | 145574   | 2096 | 1.44%   | 12.97   | 0   | 6493 | 12.00  | 17.00    | 22.00    | 67.00    | 74.69            | 374.74          | 20.13       |
| ResolveSeat | 2100     | 0    | 0.00%   | 45.46   | 29  | 202  | 45.00  | 55.00    | 60.00    | 84.99    | 1.30             | 0.86            | 0.31        |

首先看到的 Error 都是 **Response was null**, 估計可能是 JMeter 在高併發下沒有拿到 Response Body 導致, 由於 Error% = 1.44% < 2%, 就視為 false alarm。

#### ResolveSeat（掃碼進場）

* Samples：2100 = 100 + 200 + 400 + 600 + 800
* Error%：0.00% = 完全無錯誤。
* Average：**45.46 ms；p95：60 ms；p99：85 ms**。

ResolveSeat 本身需要 hit DB / visit Biz Service / push Redis，所以比 MenuRefresh 高一個量級延遲（45 ms vs 13 ms）。

但延遲仍穩定在 <100 ms，錯誤率 0%，可接受。且由於這個 API 一次 session 只呼叫一次，它的延遲稍高對體驗影響不大。

#### MenuRefresh（核心讀菜單）

* Samples：145,574 = 測試覆蓋充分。
* Error%：1.44% = 可接受範圍（SLA 通常 < 1%，但在壓測條件下 1–2% 容忍, 且判斷為 false alarm）。
* Average：12.97 ms = 平均延遲毫秒等級。
* Median (p50)：12 ms = 大多數使用者體驗接近即時。
* p95：22 ms = 95% 請求低於 22ms，符合常見 <100ms SLA 要求。
* p99：67 ms = 99% 請求低於 67ms，仍相當穩定。
* Throughput：74.69 t/s (~75 RPS)。

Redis cache 菜單 refresh 效能表現延遲在低毫秒級，即便在 800 並發下仍維持低 p95/p99，沒有明顯 tail latency。

1.44% error 需要觀察，但未必是伺服器瓶頸，可能是偶發 TCP reset 或 JMeter client GC。

整體來說，可以定義基線 SLA：p95 ≤ 25 ms，p99 ≤ 70 ms，錯誤率 < 2%。

#### 其他圖示

![Response Times Over Time](/asset/BaseLineResponseTimesOverTime.png)

![Active Threads Over Time](/asset/BaseLineActiveThreadsOverTime.png)

![Grafana](/asset/BaseLineGrafana.png)

### BaseLine 結論

#### Jmeter 報告

在 800 並發（~75 RPS） 條件下：

> MenuRefresh：p50=12 ms, p95=22 ms, p99=67 ms, Error%=1.44% 效能穩定，延遲在低毫秒級，適合作為高頻操作的 SLA 基線。

>ResolveSeat：p50=45 ms, p95=60 ms, p99=85 ms, Error%=0% 較重的初始化 API，延遲略高但仍在 100 ms 內，且錯誤率 0%，符合進場動作需求。

整體系統在現有資源配置下，穩定承載能力約為 70–80 RPS（對應 800 同時使用者，每人每 5 秒刷一次）。

#### Grafana 監控

1. QPS by Service

   * 主力流量集中在 cis-service（橘線），QPS 階梯式上升，和 JMeter 的 threads (100 → 200 → … → 800) 完全對應。
   * 最高 QPS ≈ 140 req/s，與 JMeter Aggregate Report 的 ~75 t/s 有差距，可能因為 Transaction Controller vs HTTP Sampler 數量不同，或 Prometheus 指標計算單位不同。
   * 其他服務（auth-service, order-service, store-service, gateway）都有少量流量，但整體偏低，這符合測試設計：主要壓 /cis/menu/active（透過 gateway → cis-service → Redis → store-service）。

2. p95 Latency

   * cis-service p95 ≈ 100–120 ms，相對於 JMeter 報表的 22 ms（p95），顯示 Prometheus 抓到的延遲計算包含 gateway hop 或 network overhead。
   * gateway-service p95 同步上升，這說明延遲主要來自 gateway + cis-service 路徑，沒有到達 DB 或 order-service。
   * auth-service / order-service / admin-portal 幾乎貼近 0，沒有明顯壓力，證明這次 baseline 測試主要只打到「掃碼 + 看菜單」的路徑。

3. Process CPU

   * cis-service CPU：最高也只到 3–3.5%，非常低，表示程式本身並非 CPU bound。
   * store-service CPU：也有小幅波動，因為它在後端提供菜單查詢/Redis fallback，但仍在 2% 以下。
   * 其他服務接近 idle 狀態, 看來 CPU 完全不是瓶頸。

4. Heap Usage

   * 各服務 JVM heap 用量穩定，cis-service 和 store-service 略高（1.5–2%），但完全沒逼近 GC 風險。
   * 沒有明顯的 Full GC 或 Heap 震盪。

5. Redis

   * Memory 用量：4 MiB → 幾乎不動，因為菜單資料已經緩存在 Redis，且 payload 小。
   * Ops/sec：隨 QPS 增加，峰值 ≈ 250–300 ops/s，對 Redis 來說極低。

#### 兩者比較

1. 效能觀察
   * QPS 隨並發線性上升，說明系統沒有出現瓶頸崩塌。
   * cis-service 是主要承壓點，p95 在 100–120 ms，仍遠低於常見 Web SLA（p95 < 300 ms）。
   * CPU、Heap、Redis 均極低負載 → 代表資源利用率很寬鬆。

2. 對比 JMeter 報告
   * JMeter 報的 p95 = 22 ms，Grafana 報的 p95 ≈ 100 ms → 這差異可能來自： JMeter只計 HTTP Sampler latency，不含 gateway hop / metrics overhead。

3. 穩定承載能力

   * 在 800 並發（對應 ~140 QPS），系統各服務仍處於「低資源利用率 + 穩定延遲」狀態。
   * 基線可定義：穩定承載至少 150 QPS，p95 ≈ 100 ms 以下，Error% < 2%。

4. 改進方向

   * 目前瓶頸不是 CPU/Memory/Redis，而是 cis-service 的應答延遲（100 ms），建議：
     * 確認是否包含網關序列化/反序列化開銷。
     * 增加更多分店 / 熱 key 測試，驗證 Redis 單 key 熱點行為。
     * 若要往 500–1000 QPS scale out，可考慮水平擴容 cis-service Pod。

---
