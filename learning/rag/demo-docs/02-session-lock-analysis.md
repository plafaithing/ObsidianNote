# 会话与锁分析

> DBA Copilot 知识库 · 主题 2。基于 PostgreSQL 官方文档整理，GaussDB 兼容 PG 内核，视图/函数通用。demo 用料，生产以官方手册为准。

## 概述

会话阻塞和锁等待是"数据库卡住、应用超时"的常见原因。诊断思路：**查活动会话 → 查锁等待关系 → 找阻塞源头 → 必要时终止会话**。

## 一、查活动会话

pg_stat_activity 是会话分析的核心视图。

```sql
SELECT pid, usename, datname, client_addr, state, query,
       query_start, state_change, backend_start
FROM pg_stat_activity
WHERE datname IS NOT NULL;
```

关注：
- `state = 'active'` 且 `query_start` 很久前的：长时间运行的SQL
- `state = 'idle in transaction'`：事务未提交，可能持有锁
- `client_addr`：定位是哪个应用/机器的连接

## 二、查锁等待关系

锁等待会形成"阻塞链"，要找出谁在阻塞谁。

```sql
-- 查被阻塞的会话及其阻塞源
SELECT
  blocked.pid        AS blocked_pid,
  blocked.query      AS blocked_query,
  blocking.pid       AS blocking_pid,
  blocking.query     AS blocking_query,
  blocking.state     AS blocking_state
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON pg_blocking_pids(blocked.pid) @> ARRAY[blocking.pid];
```

`pg_blocking_pids(pid)` 返回阻塞该会话的 PID 列表，是查锁等待的关键函数。

如果查询返回空，说明当前没有锁等待；返回多行，说明存在阻塞链，blocking_pid 是阻塞源头。

## 三、查锁详情（pg_locks）

看具体锁了什么对象、什么模式。

```sql
SELECT locktype, relation::regclass AS table_name, mode, granted, pid
FROM pg_locks
WHERE NOT granted;  -- 只看未获得的锁（即等待中的锁）
```

字段说明：
- `locktype`：锁类型（relation 表锁 / transactionid 事务锁 / virtualxid 等）
- `relation::regclass`：锁的表名（regclass 转换 OID 为表名）
- `mode`：锁模式（AccessShareLock 共享 / ExclusiveLock 排他 等）
- `granted`：是否已获得（false = 在等待）

常见锁模式：
- `AccessShareLock`：SELECT 时加，最轻量
- `RowExclusiveLock`：INSERT/UPDATE/DELETE 时加
- `ExclusiveLock`：排他锁，阻塞其他写

## 四、诊断步骤

1. 用户反馈"卡住"或应用超时 → 查 pg_stat_activity 看 active 会话和 idle in transaction 会话
2. 查锁等待（pg_blocking_pids）→ 定位阻塞源头 pid
3. 判断阻塞源头是否正常长事务：若 `blocking_state = 'idle in transaction'` 且长时间，异常
4. 终止阻塞源会话：`pg_terminate_backend(pid)`
5. 事后排查：为何会有长事务未提交（应用是否漏 commit）

## 五、终止会话

```sql
-- 终止指定会话
SELECT pg_terminate_backend(12345);

-- 终止某用户的所有会话
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'app_user' AND pid != pg_backend_pid();
```

注意事项：
- `pg_terminate_backend(pid)` 会回滚该会话未提交的事务
- 谨慎使用：先确认要终止的就是阻塞源，避免误杀正常业务
- `pg_cancel_backend(pid)` 只取消当前正在执行的SQL，不断开会话，更温和

## 六、常见锁等待场景

- 长事务未提交：应用执行 UPDATE 后异常未 commit，持锁导致其他会话等待
- DDL 阻塞：CREATE INDEX / ALTER TABLE 需要排他锁，阻塞读写
- 死锁：PG 自动检测并终止一方，日志有 `deadlock detected`
