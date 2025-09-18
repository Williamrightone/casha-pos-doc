# 監控與壓力測試

關於壓力測試, 會注重於找到在特定情境之下, 系統能夠運行的極限。其中比較費工的點分為事前的準備, 以及 TestCase 的定義, 這些內容都需要在執行之前有完善的計畫。

## 1. 容量/基準（Baseline + Capacity）測試

> 目標：在多分店 open model 下，Scan→Menu（讀路徑)，量出系統在固定資源上的可穩定 RPS，並給出 p50/p95/p99 SLA。

> 輸出：每一階段（10/20/40/60/80 RPS）的 p50/p95/p99、錯誤率、依賴服務指標；結論（穩定承載 RPS 與建議 SLA）。

### Usecase



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

### B. 匯出 tenants.csv

```sql
SELECT
  b.restaurant_id   AS restaurantId,
  b.id              AS branchId,
  t.qr_token        AS qrToken,
  mv.id             AS menuVersionId,
  CASE WHEN (b.branch_no % 2) = 1 THEN 5 ELSE 1 END AS weight
FROM CASHA_STORE.branch b
JOIN CASHA_STORE.table_info t     ON t.branch_id = b.id
JOIN CASHA_STORE.menu m           ON m.branch_id = b.id AND m.is_active = 1
JOIN CASHA_STORE.menu_version mv  ON mv.menu_id = m.id AND mv.status = 'ACTIVE'
WHERE b.restaurant_id = 21043822800801792
ORDER BY b.branch_no, t.table_no;
```


## Test Case

## 可能會嘗試的測試

1. 容量/基準（Baseline + Capacity）

* 目的：在多餐廳 open model下，找出在既定資源（k8s 節點、CPU、連線池、Redis、RabbitMQ）下可穩定支撐的RPS 容量，並訂 p50/p95/p99 SLA。
* 方法：Arrivals Thread Group，10→20→40→60→80 RPS 逐級階梯，每級 10–15 分鐘。
* 看：錯誤率 < 1%、p95 延遲不失控、Redis OPS/latency、RabbitMQ queue 深度、DB 連線/row lock、GC/CPU。

2. 流量混合（Traffic Mix）

* 目的：貼近真實流量，讀多寫少、多餐廳分散，含異常少量高頻餐廳。
* 方法：多 scenario/percent 配方；讀：寫 ≈ 70:30（你可用你的真實比例）。
* 看：各微服務延遲分解（Gateway → BFF → store/order），下游依賴（Redis、MQ、DB）。

3. 尖峰/突刺（Spike）

* 目的：驗證快起快落時的彈性、限流與回復速度。
* 方法：從常態 RPS（如 20）瞬間升到 200–300%（如 60），維持 2–3 分鐘，再降回常態。
* 看：MQ backlog、Redis latency、自動擴縮是否觸發（若有 HPA）、降回常態後尾延遲恢復時間。

4. 壓力/崩潰點（Stress/Break Test）

* 目的：找臨界點與優雅降級。
* 方法：每 3–5 分鐘上調 20–30% RPS，直到錯誤率 > 5% 或 p99 爆炸。
* 看：哪個服務先撐不住（熱鍵？DB？MQ？），是否有熔斷/排隊/拒絕而非全部阻塞。

5. 耐久/浸泡（Soak/Endurance）

* 目的：抓內存洩漏、連線漏、計數器飄移、TTL 不回收等慢性問題。
* 方法：以常態 RPS 跑 2–8 小時（或過夜）。
* 看：堆內存曲線、FD/Thread、連線池 borrow 時間、Redis key 增長是否回收、order.expired 死信/補償是否正常。

6. 故障注入/混沌（Chaos/Degraded Dependencies）

* 目的：確認 Redis/MQ/DB 降速或瞬斷時，應用是否快速失敗、降級、重試退避、最終一致。
* 方法：在壓測途中調慢 Redis/MQ/DB，或限流某個服務；觀察重試、熔斷開合、補償隊列堆積。
* 看：order.payment.await → order.expired 延時/補償是否準確，是否產生重複扣庫或遗漏釋放。