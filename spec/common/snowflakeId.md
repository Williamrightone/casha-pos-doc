# SnowflakeId

本專案 採用 Snowflake ID 唯一 ID 生成服務，基於 Twitter Snowflake 演算法改進實現。

組成結構為:
> [時間戳(42)] [數據中心ID(5)] [機器ID(5)] [序列號(12)]

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