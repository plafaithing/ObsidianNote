# 连接管理与池化

> DBA Copilot 知识库 · 主题 3。基于 PostgreSQL 官方文档 + PgBouncer 文档整理，GaussDB 兼容 PG 内核。demo 用料，生产以官方手册为准。

## 概述

PostgreSQL 每个连接是一个独立进程，连接数多了内存和上下文切换开销大。连接耗尽会报 `too many connections`。管理思路：**监控连接数 → 处理 idle 连接 → 用连接池复用连接**。SQL 限流在 PG 里主要通过连接数控制 + 连接池实现。

## 一、连接数监控

```sql
-- 当前总连接数
SELECT count(*) AS total_connections FROM pg_stat_activity;

-- 各用户连接数分布
SELECT usename, count(*) AS conn_count
FROM pg_stat_activity
WHERE datname IS NOT NULL
GROUP BY usename
ORDER BY conn_count DESC;

-- 各数据库连接数
SELECT datname, count(*) FROM pg_stat_activity
WHERE datname IS NOT NULL GROUP BY datname;
```

## 二、max_connections 参数

```sql
SHOW max_connections;
-- 调整（需重启数据库生效）
ALTER SYSTEM SET max_connections = 200;
```

建议：
- 纯 OLTP 配合连接池：100-300
- 不要盲目调大：每个连接有内存开销（几MB），连接太多会导致内存压力和上下文切换
- 配合 PgBouncer 时，PG 侧 max_connections 可设小，PgBouncer 侧复用

## 三、idle 连接问题

长时间 idle 的连接占用 PG 进程资源，是连接数虚高的主因。

```sql
-- 查 idle 超过 1 小时的连接
SELECT pid, usename, client_addr, state, state_change,
       now() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle'
  AND state_change < now() - interval '1 hour';
```

idle 连接多的原因：
- 应用连接池配置过大，开了连接不用
- 应用异常退出未关闭连接
- 长连接服务未做连接回收

## 四、PgBouncer 连接池

PgBouncer 是轻量级连接池，在应用和 PG 之间复用连接，大幅减少 PG 进程数。

工作模式：
- `transaction pooling`（事务级复用，推荐）：事务结束后连接归还池，复用率高
- `session pooling`（会话级复用）：会话结束才归还，复用率低
- `statement pooling`（语句级）：不适用有事务的场景

关键配置（pgbouncer.ini）：
```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
pool_mode = transaction
default_pool_size = 25       ; 每个 db+user 的服务端连接数
max_client_conn = 1000       ; 客户端最大连接数
```

效果：客户端可建 1000 连接，PG 侧只有 25 个服务端连接在复用，PG 压力大减。

## 五、诊断步骤

1. 报 `too many connections` → 查总连接数 vs max_connections
2. 连接数高但 active 少 → idle 连接堆积 → 排查应用连接池配置
3. 短期缓解：终止 idle 连接 `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle' AND ...`
4. 长期方案：上 PgBouncer 连接池，PG 侧 max_connections 不用设太大

## 六、连接数与"限流"的关系

PG 原生没有"SQL 限流"功能（如限制某用户 QPS）。限流思路：
- **连接数控制**：通过 max_connections + PgBouncer pool_size 限制某用户最大并发，间接限流
- **应用侧限流**：在应用层做 QPS 控制
- GaussDB 部分版本提供 SQL 限流功能（具体语法见各版本手册）

即：PG 的"限流"主要靠连接数管控，不是原生 QPS 限流。
