# 参数调优

> DBA Copilot 知识库 · 主题 4。基于 PostgreSQL 官方文档整理，GaussDB 兼容 PG 内核，参数通用。demo 用料，生产以官方手册为准。

## 概述

PG 关键内存参数对性能影响大。调优核心四个参数：**shared_buffers / work_mem / effective_cache_size / maintenance_work_mem**。原则：先看缓存命中率定位是否内存问题，再针对性调参。

## 一、shared_buffers（共享缓冲区）

PG 用来缓存数据页的共享内存，是最关键的内存参数。

```sql
SHOW shared_buffers;
-- 调整（需重启生效）
ALTER SYSTEM SET shared_buffers = '4GB';
```

建议值：**物理内存的 25%**。例如 16GB 内存的服务器，shared_buffers 设 4GB。

不要设过大：超过 25% 收益递减，且 PG 还要依赖操作系统页缓存。

## 二、work_mem（工作内存）

单条SQL做排序（ORDER BY）或哈希（HASH JOIN）时使用的内存。调大可减少落盘。

```sql
SHOW work_mem;
-- 调整（reload 生效，无需重启）
ALTER SYSTEM SET work_mem = '64MB';
SELECT pg_reload_conf();
```

建议值：64MB - 256MB。注意：`work_mem 是每个操作每个连接独立分配`，总内存占用 ≈ work_mem × 并发操作数。设太大在并发高时会爆内存。

诊断：如果 EXPLAIN ANALYZE 看到 `Sort Method: external merge Disk`，说明排序落盘，要加大 work_mem。

## 三、effective_cache_size（有效缓存预估）

告诉查询优化器"系统大约有多少内存可用于缓存"，影响执行计划选择（是否选索引扫描）。

```sql
SHOW effective_cache_size;
-- 调整（reload 生效）
ALTER SYSTEM SET effective_cache_size = '12GB';
```

建议值：**物理内存的 50% - 75%**。

注意：这个参数**不分配实际内存**，只是给优化器一个估值。设小了优化器会误以为内存少、倾向全表扫描；设大了优化器更倾向用索引。

## 四、maintenance_work_mem（维护内存）

VACUUM、CREATE INDEX、ALTER TABLE 等维护操作用的内存。调大可加速这些操作。

```sql
SHOW maintenance_work_mem;
-- 调整（reload 生效）
ALTER SYSTEM SET maintenance_work_mem = '1GB';
```

建议值：1GB - 2GB。因为维护操作通常不并发，可以设大些，加速 VACUUM 和建索引。

## 五、诊断：看缓存命中率

调参前先看命中率，判断是否内存瓶颈。

```sql
-- 查各数据库的缓存命中率
SELECT
  datname,
  blks_hit,
  blks_read,
  round(blks_hit::numeric / nullif(blks_hit + blks_read, 0) * 100, 2) AS hit_ratio_pct
FROM pg_stat_database
WHERE datname IS NOT NULL;
```

判断：
- `hit_ratio_pct > 99%`：缓存命中良好，内存参数基本合理
- `hit_ratio_pct < 95%`：命中率低，可能 shared_buffers 不够或有全表扫描SQL

## 六、调优步骤（评估驱动）

1. **看命中率**（pg_stat_database）→ 定位是否内存问题
2. **shared_buffers 先调**：设到物理内存 25%
3. **effective_cache_size**：设到物理内存 50-75%（让优化器选对计划）
4. **work_mem**：看是否有排序落盘（EXPLAIN ANALYZE），逐步调大
5. **maintenance_work_mem**：设 1-2GB，加速 VACUUM/建索引
6. **每次只改一个参数，测效果**（评估驱动，对应 L09：改一个变量跑评估对比）

## 七、其他常见参数

- `max_connections`：最大连接数（见"连接管理"文档）
- `wal_buffers`：WAL 日志缓冲，默认 -1（auto，通常够用）
- `random_page_cost`：随机读成本，SSD 上可调小到 1.1（默认 4.0 偏向顺序扫描）
- `checkpoint_completion_target`：检查点平滑度，建议 0.9

## 八、ALTER SYSTEM 说明

`ALTER SYSTEM SET` 写入 postgresql.auto.conf，优先级高于 postgresql.conf。改完：
- reload 生效的参数（如 work_mem）：`SELECT pg_reload_conf();`
- 需重启的参数（如 shared_buffers、max_connections）：重启数据库

查看参数是否需重启：`pg_settings` 视图的 `context` 字段（`postmaster` 表示需重启，`user`/`superuser` 表示 reload 或会话级生效）。
