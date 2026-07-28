# HNSW 速查表

> 向量库索引算法速查。RAG 课程 L05 / Q&A 0007 / 期中 Q11-Q12 配套。售前方向，重取舍不重算法实现。

## 是什么

HNSW（Hierarchical Navigable Small World，分层可导航小世界图）= 向量库的**图索引**。

- 不建索引 = 全表扫描（flat），N 个向量算 N 次距离，慢
- HNSW 把向量组织成多层图，查询时从顶层粗定位、逐层下沉，跳着找 → 快
- 代价：近似（可能漏个别最近邻），但 recall 可调到 95%+

**DBA 类比**：HNSW 之于向量库 = B-Tree 之于关系库。flat = 全表扫描。

## 三参数

| 参数 | 什么时候定 | 作用 | 默认值 | DBA 类比 | 改动成本 |
|---|---|---|---|---|---|
| **M** | 建索引时 | 每个向量的邻居连接数（图密度）。大→密→recall高→内存大 | 16-32（BGE-M3 常用 30）| 索引 fanout / 页大小 | 大（重建图结构）|
| **efConstruction** | 建索引时 | 建图时搜多宽。大→图质量好→建图慢 | 200-400（BGE-M3 常用 360）| 建索引的统计采样精度 | 中（重建图）|
| **ef**（ef_search）| **查询时** | 查询时搜多宽。大→recall高→查询慢 | 检索时设（64-128）| 查询 hint（扫描行数）| 零（查询参数）|

## 调参顺序（成本递增）

```
ef（零成本）→ efConstruction（重建图）→ M（重建图结构）→ 退回查上游
```

1. **先调 ef**：只改查询参数，不动索引。加大 ef 看 recall 涨不涨。最便宜。
2. **再调 efConstruction**：要重建图，但向量不用重新 embed。中成本。
3. **最后调 M**：要重建图结构。大成本。
4. **都不够** → 问题在上游：维度灾难、切分太碎、embedding 选型。回查 L05。

**口诀**：能调查询参数就别重建索引，能重建索引就别换模型。和 DBA 调优一个思路——先 hint，再重建索引，最后改表结构。

## 维度对 HNSW 的三重影响（Q&A 0007）

维度从 1024 升到 3072，不只"存储翻 3 倍"：

| 影响 | 说明 |
|---|---|
| **内存** | 向量存储线性翻 3 倍 + HNSW 图开销跟涨 |
| **延迟** | 距离计算 O(维度)，线性涨 |
| **召回（维度灾难）** | 高维空间稀疏，同 M 下邻居覆盖不足，recall 掉；要保 recall 得加大 M/ef，成本雪上加霜 |

**结论**：真实代价比"存储翻 3 倍"大得多。1024 是运维甜点，别盲目追高维。

## 症状 → 调参诊断表

| 症状 | 根因 | 动作 |
|---|---|---|
| recall 不够，延迟还有余量 | ef 太小 | 加大 ef（最便宜）|
| 加 ef recall 不涨了 | 图质量差 | 加大 efConstruction 重建 |
| 还不够 | 图太稀疏 | 加大 M 重建 |
| 都调到位还是不够 | 上游问题 | 查维度 / 切分 / embedding 选型 |
| recall 够但延迟超标 | ef 太大 | 降 ef，或换更小维度模型 |
| 建索引慢 | efConstruction 大 | 接受（一次性成本）或适当降 |

## DBA 索引类比总表

| HNSW 概念 | DB 类比 |
|---|---|
| 不建索引（flat）| 全表扫描 |
| HNSW 图索引 | B-Tree 索引 |
| M（连接数）| 索引 fanout / 页大小 |
| efConstruction | 建索引的统计采样精度 |
| ef（ef_search）| 查询 hint（扫描行数）|
| 调 ef | 改 hint（不动索引）|
| 调 efConstruction | 重建索引 |
| 调 M | 改索引结构 |
| 维度灾难 | 字段过长导致索引膨胀 / 低效 |

## 配置示例（伪代码）

```python
# Milvus / Qdrant 风格，参数含义通用
collection.create_index(
    field="vector",
    index_type="HNSW",
    params={
        "M": 30,                # 建索引时定，BGE-M3 常用 30
        "efConstruction": 360,  # 建索引时定，BGE-M3 常用 360
    }
)

# 查询时调 ef（不用重建索引）
results = collection.search(
    query_vector=q,
    param={"ef": 128},   # 检索时设，recall 不够先调这个
    limit=5
)
```

## 记忆口诀

- **三参数**：M 建图定密度，efConstruction 建图定质量，ef 查询定广度
- **调参顺序**：ef → efConstruction → M → 上游
- **维度**：内存 + 延迟 + 维度灾难，1024 是甜点

---
配套：[L05 Embedding 选型](../lessons/0005-embedding-selection.html) · [Q&A 0007](../Q&A.md) · [期中考试 Q11-Q12](../quizzes/midterm.html)
