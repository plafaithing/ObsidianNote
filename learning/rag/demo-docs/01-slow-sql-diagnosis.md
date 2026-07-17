# 慢SQL诊断

> DBA Copilot 知识库 · 主题 1。基于 PostgreSQL 官方文档整理，GaussDB 兼容 PG 内核，视图/参数通用。demo 用料，生产以官方手册为准。

## 概述

慢SQL是数据库性能下降最常见的原因。诊断思路：**发现慢SQL → 定位执行计划瓶颈 → 优化（加索引/重写SQL/调参数）**。PostgreSQL/GaussDB 提供三件套：慢SQL日志、pg_stat_statements、pg_stat_activity。

## 一、开启慢SQL日志

`log_min_duration_statement` 参数记录执行超过阈值的SQL，是发现慢SQL的第一道关口。

```sql
-- 记录执行超过 1 秒的 SQL（单位：毫秒）
ALTER SYSTEM SET log_min_duration_statement = 1000;
SELECT pg_reload_conf();
```

设置后，日志里会看到类似：`duration: 1234.567 ms  statement: SELECT * FROM orders WHERE ...`。设为 0 记录所有SQL（调试用，生产慎用），设为 -1 关闭。

## 二、用 pg_stat_statements 看历史慢SQL

pg_stat_statements 扩展记录所有SQL的执行统计，是找"累计最耗时SQL"的利器。

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 查累计最耗时的 10 条 SQL
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

字段说明：
- `query`：SQL 文本
- `calls`：执行次数
- `total_exec_time`：总执行时间（PG 13+；PG 12 及之前为 `total_time`）
- `mean_exec_time`：平均执行时间（`total_exec_time / calls`）
- `rows`：返回/影响行数

关注点：`mean_exec_time` 高的单次慢SQL；`calls` 高的累计耗时SQL；`rows` 远大于预期的可能是缺失过滤条件。

## 三、用 pg_stat_activity 看当前正在执行的SQL

pg_stat_activity 看实时会话，找"正在卡住"的SQL。

```sql
SELECT pid, usename, datname, state, query, query_start, state_change
FROM pg_stat_activity
WHERE state != 'idle';
```

state 取值：
- `active`：正在执行SQL
- `idle`：空闲，等待新请求
- `idle in transaction`：事务中空闲（需重点关注，可能持有锁未释放）
- `idle in transaction (aborted)`：事务中某语句出错，仍持有事务

`query_start` 与当前时间差大 + state=active，说明这条SQL执行了很久。

## 四、诊断步骤

1. **发现慢**：慢SQL日志或 pg_stat_statements 找出耗时SQL
2. **看执行计划**：`EXPLAIN ANALYZE` 查看实际执行路径
3. **定位瓶颈**：全表扫描？索引缺失？嵌套循环选错？排序落盘？
4. **优化**：加索引 / 重写SQL / 调参数（如 work_mem）

## 五、执行计划分析

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;
```

重点看节点类型：
- `Seq Scan`（全表扫描）：大表上出现通常要加索引
- `Index Scan` / `Index Only Scan`（索引扫描）：正常
- `Sort` 节点带 `Sort Method: external merge`：排序落盘了，加大 work_mem
- `Rows Removed by Filter` 很大：过滤性差，考虑索引或改查询

## 六、常见优化手段

- 缺索引：对 WHERE / JOIN / ORDER BY 字段加 B-Tree 索引
- 统计信息过期：`ANALYZE 表名` 更新统计信息，让优化器选对计划
- 排序落盘：加大 work_mem
- 大表全表扫描：考虑分区表或覆盖索引
