# SnowflakeId

## Primary Key

使用 Snowflake ID 來確保全局唯一且趨勢遞增，避免自增 ID 對分庫分表的限制；同時引入 shard_code 作為路由維度，應用層計算分片(ShardingSphere)，避免數據庫層 FK 造成瓶頸

部分資料量較少的表格如 RBAC 相關則不使用分片

| 1 bit |    41 bits    |   10 bits   |  12 bits   |
| ------ | -------| -------| ------- |
| 符號位 |  時間戳(ms)   | 機器ID      | 序列號      |

> 起始時間 2025-07-22 00:00:00 UTC = 1753056000000
> snowflake.atacenter-id=1 # 資料中心 ID (1~2) 
> datacenter-id: 1   # 資料中心 ID (1~2 預設台北高雄各一台)
> machine-id: 3      # 機器 ID (1~2 預設每個 DC 各一台 VM)

```java
@Component
public class SnowflakeIdGenerator {

    // 起始時間戳 (Epoch 設定為 2025-07-22)
    private static final long EPOCH = 1753056000000L;

    // 位元長度設定
    private static final long DATA_CENTER_BITS = 5L;
    private static final long MACHINE_BITS = 5L;
    private static final long SEQUENCE_BITS = 12L;

    // 位移數
    private static final long MACHINE_SHIFT = SEQUENCE_BITS;
    private static final long DATA_CENTER_SHIFT = SEQUENCE_BITS + MACHINE_BITS;
    private static final long TIMESTAMP_SHIFT = SEQUENCE_BITS + MACHINE_BITS + DATA_CENTER_BITS;

    // 最大值
    private static final long MAX_SEQUENCE = ~(-1L << SEQUENCE_BITS);

    @Value("${snowflake.datacenter-id:1}")
    private long dataCenterId;

    @Value("${snowflake.machine-id:1}")
    private long machineId;

    private long sequence = 0L;
    private long lastTimestamp = -1L;

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("時鐘回撥，拒絕產生 ID");
        }

        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }

        lastTimestamp = timestamp;

        return ((timestamp - EPOCH) << TIMESTAMP_SHIFT)
                | (dataCenterId << DATA_CENTER_SHIFT)
                | (machineId << MACHINE_SHIFT)
                | sequence;
    }

    private long waitNextMillis(long lastTimestamp) {
        long timestamp = System.currentTimeMillis();
        while (timestamp <= lastTimestamp) {
            timestamp = System.currentTimeMillis();
        }
        return timestamp;
    }

}
```

## Foreign Key

在 MySQL 裡不強制設定 FK，而是應用層驗證，因為專案走微服務架構，避免 FK 造成分庫困難，避免寫入性能瓶頸。

## JPA & Native Query

簡單 CRUD 場景用 JPA，在大量聚合和批量計算場景，用 Native SQL

## Flyway DB

專案使用 Flyway DB, DB 命名為大寫 + 底線如 ```AUTH_DB```,  table 則是小寫 + 底線, 如 ```teble_info```

在 Flyawy 的管理命名上:

Version 從 1.0.0 開始, 每次更新尾數 +1, 經大版本則中間數 +1

* 若更新 DDL, 須以 create, alter 開頭
* 若更新 DML, 須以 insert 開頭

```sql
# V1.0.0__create_teble_casha_user.sql
# V1.0.1__create_table_role.sql
...
# V1.0.6__alter_table_function_menu_add_column_permission_code.sql
```

