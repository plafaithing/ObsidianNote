# RAG 课程 Q&A

> 用户提问的概念/机制类问题沉淀，持续追加。方便后续 Obsidian 复习。
> 编号 `0001-xxx`，按提问顺序递增。

---

## 0001 · 向量检索（⑤）可能出现哪些问题症状？

**定位**：向量化（③）是"把文本映射到向量空间"（画地图），检索（⑤）是"在向量空间里找最近邻"（在地图上找）。地图画对了但找的方法不对，就是检索问题。检索环节症状的本质是：**在正确的向量空间里，却搜不到 / 搜不全 / 搜得乱**。

**典型症状**：

| 症状 | 原因 | 运维类比 | 解法 |
|---|---|---|---|
| 相关 chunk 没进候选，答案残缺 | K 太小 | `LIMIT N` 设太小漏数据 | 调大 K（配合 rerank） |
| 库里明明有相关文档，Top-K 却没有 | ANN 近似损失（HNSW `ef_search` 太低） | 索引建太稀疏，走索引漏数据 | 调大 `ef_search` |
| 问精确标识符/错误码检索不到 | 纯向量对精确匹配弱 | 全文搜不到精确字符串 | 混合检索（向量+BM25） |
| 检索结果排序不符直觉 | 相似度度量选错（cosine/L2/dot） | 排序规则不对 | 多数用 cosine |
| 新文档查不到 / 旧文档还返回 | 索引未同步 | 新分区没建索引 / 删数据索引没更新 | 增量索引更新策略 |

**三个最该记住的**：

1. **ANN 近似损失**：HNSW 是近似最近邻，`ef_search` 太低会漏掉真正相关文档。和 embedding 选错的区别——embedding 错是"大面积跑偏 + 跨语言/领域"，ANN 错是"地图对但搜不全"。
2. **纯向量对精确匹配弱**（最重要，L06 混合检索的根本动机）：向量检索是"语义相似"，对视图名（`PGXC_STAT_ACTIVITY`）、错误码（`ORA-01555`）、参数名等精确标识符弱。BM25 补精确匹配。真实系统几乎都用混合检索。
3. **K 值**：太小漏召回，太大涌入噪声 + 增 rerank 成本。常见做法 K 设大（20-50）广召回，rerank 精排到 Top-3~5。

**区分向量化 vs 检索口诀**：
- 召回**大面积跑偏** + 语种/领域特殊 → **向量化**（地图画错了）
- 地图对但**搜不到/搜不全/搜得乱** → **检索**（找的方法不对）

**关联课程**：[L02 全流程地图](lessons/0002-rag-pipeline.html) · [L02 排查练习](lessons/0002-practice-troubleshooting.html) · [L06 混合检索（待学）](lessons/0006-hybrid-retrieval.html)

---

## 0002 · Recall@K 是什么？检索评估指标家族

**一句话**：Recall@K = 检索返回的 Top-K 结果中，包含了多少个「真正相关」的文档。公式：`Top-K 中相关数 ÷ 全部相关文档数`。

**关键前提**：得先有「标准答案」（ground truth）——人工标注的「这个查询的相关文档是哪几个」。没标注就算不了 Recall。

**数字例子**：库里和「慢SQL诊断」相关的文档有 10 个（ground truth），检索 Top-5 里有 3 个相关：
- **Recall@5 = 3/10 = 30%**（10 个相关里 Top-5 只捞到 3 个 → 捞全了 30%）
- **Precision@5 = 3/5 = 60%**（Top-5 里有 3 个相关 → 捞准了 60%）

**Recall vs Precision（核心：分母不同）**：

分子相同（都是「Top-K 中相关数」），分母不同——这是易混根源：

```
库里相关文档：10 个（ground truth）
检索 Top-5：返回 5 个，其中相关 4 个

Recall@5    = 4 / 10 = 40%   ← 分母=全部相关（该捞的）
Precision@5 = 4 / 5  = 80%   ← 分母=K（捞回的）
```

- **Recall@K**：捞全没（分母=全部相关文档）→ 该捞的捞回了多少比例
- **Precision@K**：捞准没（分母=K）→ 捞回的有多少比例是相关的

**钓鱼类比**：池塘 10 条目标鱼，撒网捞上 5 条其中 4 条是目标鱼。Recall=4/10（该捞的漏 6 条），Precision=4/5（捞得很准没捞杂鱼）。

**优化口诀：漏了多捞，杂了精排**：
- Recall 低 = 漏了 = **多捞**（增大 K / 混合检索 / 调 ANN `ef_search`）
- Precision 低 = 杂了 = **精排**（加 rerank，提 precision 不提 recall）

**rerank 提 Precision 不提 Recall**：rerank 只对已召回的 Top-K 重排序，不增加召回数量。漏掉的文档不在 Top-K 里，rerank 排不出来。所以 Recall 低别急着加 rerank，先多捞。

**为什么 RAG 特别看 Recall**：
- **漏召回** = 答案在库里但没喂给 LLM → 答案残缺/幻觉（致命）
- **多召回** = Top-K 混入噪声 → 可靠 rerank 降权、LLM 忽略（危害小）
- 所以 recall 是底线，precision 靠 rerank 补。这正是 L07 rerank 68%→89% 的逻辑：先 recall 广召回（K 设大 20-50），再 rerank 精排到 Top-3~5 提精度。

**@K 指标家族**：

| 指标 | 含义 | 关注点 |
|---|---|---|
| Recall@K | Top-K 中相关数 ÷ 全部相关数 | 捞全没 |
| Precision@K | Top-K 中相关数 ÷ K | 捞准没 |
| Hit Rate@K | Top-K 里至少命中一个相关的查询占比 | 命中没 |
| MRR | 第一个相关结果的排名倒数（1/rank） | 排多前 |
| NDCG@K | 综合排序质量（相关越靠前分越高） | 排序好不好 |

**K 怎么选**：RAG 通常看 Recall@3 或 Recall@5（最终喂给 LLM 的就是 Top-3~5）。Recall@1 = 第一个结果就命中相关的概率。

**运维类比**：Recall@K 像 SQL 查询的查全率。库里符合条件 10 条，Top-5 返回 3 条符合 → Recall@5 = 30%。区别：数据库精确 `WHERE` 通常 100% 召回（条件对就全捞回），RAG 向量检索是「近似相似度」会漏——所以才要测 Recall@K 评估漏多少。这也是 L06 上混合检索（向量+BM25 互补提 recall）的根本动机。

**关联课程**：[L02 全流程地图](lessons/0002-rag-pipeline.html)（评估环节）· [L03 切分策略](lessons/0003-chunking-strategies.html)（评测驱动定 chunk 大小）· [L07 Rerank（待学）](lessons/0007-rerank.html)（recall 广召回 + rerank 精排）

---

## 0003 · 元数据抽取是怎么操作的？和数据库 ETL 什么关系？

**本质**：概念上是 ETL（抽-转-装），但不是对数据库表做，而是 **RAG Ingest 管道的一步**——切分后、入库前给每个 chunk 贴标签。元数据 ingest 时一次性抽好、和向量一起存进向量库，检索时直接拿来过滤。

**管道位置**：`原始文档 → 解析 → 切分(chunk) → 【元数据抽取】 → 向量化 → 入库(向量+元数据)`

**两种抽取方式**：

**① 规则抽取（优先，免费零误差）**——从文档自带结构读：
- 路径：`docs/gaussdb/v8.0/慢SQL.md` → product=gaussdb, version=v8.0
- 标题：Markdown `##` → section
- frontmatter：YAML 头 → 直接读
- 文件名：`install-guide.md` → doc_type=安装手册

代码示意（Python）：
```python
from pathlib import Path
def extract_metadata(chunk_text, file_path):
    p = Path(file_path); parts = p.parts; meta = {}
    if len(parts) >= 3:
        meta['product'] = parts[1]; meta['version'] = parts[2]
    name_map = {'install': '安装手册', 'troubleshoot': '故障排查'}
    meta['doc_type'] = next((v for k,v in name_map.items() if k in p.stem.lower()), '其他')
    meta['section'] = chunk_text.split('\n')[0] if chunk_text else ''
    meta['source_url'] = f"https://wiki.example.com/{p.as_posix()}"
    return meta
# 切分后每个 chunk 调一次，连同向量一起存入向量库
```

**② LLM 抽取（兜底，非结构化文档）**——扫描件 OCR、Word 纯文本等没结构信息的文档，让 LLM 读 chunk 抽 JSON。要点：用 JSON mode 保证可解析；加白名单校验（product 不在白名单标「未知」）；成本高（每 chunk 调一次 LLM），只在规则搞不定时用。

**和数据库 ETL 对比**：

| | 数据库 ETL | RAG 元数据抽取 |
|---|---|---|
| 数据源 | 关系库表（结构化） | 文档（非结构化/半结构化） |
| 抽取对象 | 表字段 | 路径/标题/frontmatter/chunk 内容 |
| 转换 | 字段映射/清洗/聚合 | 规则解析 / LLM 抽取 |
| 装载目标 | 数据仓库 | 向量库 payload 字段 |
| 触发 | 批处理 / 增量 | 文档入库时（一次性+增量） |
| 工具 | Airflow / DataX / dbt | LangChain / LlamaIndex / Dify 内置 |

相似：都是「抽-转-装」，都讲究增量同步、字段一致性、校验。不同：ETL 处理结构化表字段，RAG 抽取处理文档结构信息或靠 LLM 从内容抽。

**Dify**：内置基本元数据（来源、分段号），自定义字段（product/version）靠按目录组织上传（路径即元数据）或 API 自定义 ingest 管道。L10 动手时具体演示。

**工程化注意**：①抽取时机（ingest 一次抽好存库）；②增量更新（文档更新重新抽取）；③一致性（字段名/值域白名单与检索过滤条件对齐）；④校验（规则处理路径不规范，LLM 校验输出格式）。

**关联课程**：[L04 元数据与结构化抽取](lessons/0004-metadata-extraction.html)（why + 字段设计）· [L10 Dify demo（待学）](lessons/0010-dify-demo.html)（Dify 实操）

---

## 0004 · 768 / 1024 / 1536 / 3072 是什么？embedding 维度详解

**一句话**：这些数字是 embedding 向量的**维度**——一段文本被编码后输出的那串数字有几个分量。1024 维 = 输出 1024 个数字的数组 `[0.12, -0.45, ..., 0.03]`。

**直观理解**：2 维是平面坐标 (x,y)，3 维是空间坐标 (x,y,z)，1024 维是 1024 个「语义特征轴」从 1024 个角度刻画文本语义。维度越高，语义细节越精细，区分力越强。

**数据库类比**：向量库里每个 chunk ≈ 一行记录，**维度 ≈ 列数**。1024 维 = 每个向量有 1024 个「特征列」。区别：DB 列是人定义的字段（语义明确），embedding 的「列」是模型学出来的语义特征（不可解释——你不知道「维度 537」代表什么，但模型用它区分语义）。

**数字哪来的**：模型架构决定，不是可调参数。选了 BGE-M3 就是 1024 维，OpenAI text-embedding-3-large 就是 3072 维。能选「用哪个维度的模型」，不能「给模型调维度」。常见值：bge-base-zh=768，BGE-M3/bge-large-zh=1024，OpenAI small=1536，OpenAI large=3072。

**维度高低的权衡**：高维表达力强但存储大/检索慢（存储=维度×4字节×向量数，100万×1024×4B≈4GB）；低维省资源但可能丢语义细节。**1024 是甜点**，运维场景够用且成本可控，没必要上 3072（成本翻3倍，效果边际递减）。

**关联课程**：[L05 Embedding 选型](lessons/0005-embedding-selection.html)（维度权衡表）

---

## 0005 · BGE-M3 已支持稀疏向量，为什么还要 BM25？会重复吗？

**核心误解**：BGE-M3 稀疏向量 ≠ BM25，看着都叫"稀疏"，本质完全不同。

**本质区别**：
- **BM25**：纯字面统计（TF-IDF 改进），算法"算"出来，**不依赖模型**。文档出现 `ORA-01555` 字符串就能匹配，算的是词频/文档频率/文档长度归一化。
- **BGE-M3 稀疏向量**：模型**学习**出来的神经稀疏（neural sparse），给每个 token 学权重。比 BM25 聪明（有点语义），但**依赖训练语料**——没见过的 token 权重学不好。

**对比**：

| | BM25 | BGE-M3 稀疏 |
|---|---|---|
| 本质 | 字面统计（算） | 神经稀疏（学） |
| 依赖模型 | 不依赖 | 依赖（跑推理） |
| 罕见标识符/未登录词 | ⭐强（不在乎见没见过） | 弱（没见过学不好） |
| 语义能力 | 无 | 有一点（不如稠密强） |
| 可解释性 | ⭐强 | 弱（黑盒） |
| 成本 | 零训练 | 要跑模型 |

**为什么不重复**：常见词上重叠，但不重叠两块——①罕见标识符（ORA-01555）：BM25 字面匹配最稳，BGE-M3 稀疏没见过学不好；②语义关联：BGE-M3 稀疏比 BM25 强一点，但远不如它自己的稠密向量。

**三种架构**：
- **一站式**：BGE-M3（稠密+稀疏）不要 BM25——省事，文档量中等够用
- **经典混合**：稠密（任意模型）+ BM25——⭐最常见，复用 ES/OpenSearch，BM25 最稳
- **三路融合**：少见，BGE-M3 稀疏和 BM25 重叠多，性价比低

**L05 说 BGE-M3「一站式做混合检索」= 架构1**，一个模型同时出稠密+稀疏，省掉维护 BM25 索引。不是说它稀疏=BM25，而是自带"类 BM25"能力，很多场景够用。

**运维场景**：标识符多且罕见（错误码/视图名/参数名）→ 经典混合（稠密+BM25）最稳；想简化架构 → BGE-M3 一站式够用；不确定 → 评测对比 Recall@5。

**一句话**：BM25 是"算"的字面统计，BGE-M3 稀疏是"学"的神经稀疏。经典混合检索（稠密+BM25）至今主流，因为 BM25 对罕见标识符更稳、可解释、零模型成本。BGE-M3 一站式是"省事"不是"更强"。

**关联课程**：[L05 Embedding 选型](lessons/0005-embedding-selection.html)（BGE-M3 三合一）· [L06 混合检索（待学）](lessons/0006-hybrid-retrieval.html)

---

## 0006 · BGE-M3 三合一具体怎么用？

**M3 三个 M**：Multi-Functionality（dense+sparse+colbert 三种检索）+ Multi-Linguality（100+ 语言）+ Multi-Granularity（最长 8192 tokens）。核心价值：**一次 `encode()` 同时出三种向量**，sparse 零额外成本（官方原话："obtain token weights similar to BM25 without any additional cost"）。

**三种功能**：

| 功能 | 输出 | 维度 | 检索方式 | 擅长 | 成本 |
|---|---|---|---|---|---|
| Dense（稠密） | 1 个向量 | 1024 | cosine/点积 | 语义相似 | 低 |
| Sparse（稀疏/词法权重） | 稀疏向量 | 250002（xlm-roberta 词表） | 词法匹配（类 BM25） | 精确 token | 低（只存非零） |
| ColBERT（多向量） | 每 token 1 向量 | N×1024 | MaxSim（late interaction） | 精度最高 | ⚠️ 高 |

**ColBERT 的 MaxSim**：query 每 token 和 doc 所有 token 算相似度取最大，再求和。token 级精细匹配，精度最高但存储爆炸（每 chunk 存几十上百个向量）。

**代码（FlagEmbedding）**：
```python
from FlagEmbedding import BGEM3FlagModel
model = BGEM3FlagModel('BAAI/bge-m3', use_fp16=True)
output = model.encode(texts,
    return_dense=True, return_sparse=True, return_colbert_vecs=True)
# dense: output['dense_vecs']  1024 维
# sparse: output['lexical_weights']  {token_id: weight}，用 compute_lexical_matching_score 算分
# colbert: output['colbert_vecs']，用 colbert_score 算 MaxSim
```
**关键**：BGE-M3 **不再需要给 query 加 instruction**（区别于 bge-v1.5）。

**Milvus 集成**：`BGEM3EmbeddingFunction`，`encode_documents()` / `encode_queries()`，返回 `{'dense':..., 'sparse': csr_array}`。

**三路融合**：加权求和 `compute_score(weights_for_different_modes=[w_dense, w_sparse, w_colbert])`，模型内部归一化（三种分量纲不同）。官方推荐 pipeline：**hybrid retrieval（dense+sparse）+ reranking**（bge-reranker-v2）。

**运维场景选法**：
- 只要 dense：最省，精确标识符弱
- **dense + sparse：最常见**，互补，存储可控
- 三路全用：ColBERT 太贵，留给高精度场景（法律/医疗），运维 RAG 很少用

**和 Q&A 0005 的衔接**：BGE-M3 sparse 是"学"的神经稀疏（不是 BM25 的字面统计），对罕见标识符仍不如 BM25 稳——所以经典混合（dense+BM25）仍是主流。BGE-M3 一站式（dense+sparse）是"省事"不是"更强"。

**关联课程**：[L05 Embedding 选型](lessons/0005-embedding-selection.html)（BGE-M3 推荐）· [L06 混合检索（待学）](lessons/0006-hybrid-retrieval.html)（融合机制）

---

## 0007 · embedding 维度和 HNSW 参数什么关系？

**HNSW 是什么**：向量库默认的 ANN（近似最近邻）索引，多层小世界图。检索=顶层入口→贪婪搜索→层层下降→底层细化。和 B-tree 类比：B-tree 精确（100% 召回），HNSW 近似（有 recall 损失）——所以 HNSW 才谈 recall@K，B-tree 不谈。这正是 Q&A 0001 的"ANN 近似损失"。

**三个核心参数**（核实自 Milvus v2.6 官方文档）：

| 参数                 | 阶段  | 含义             | 默认  | 推荐       | 调大         |
| ------------------ | --- | -------------- | --- | -------- | ---------- |
| **M**              | 建索引 | 每节点最大连接数（出+入边） | 30  | [5,100]  | 召回↑内存↑建索引慢 |
| **efConstruction** | 建索引 | 建索引时候选邻居数      | 360 | [50,500] | 图质量↑建索引慢   |
| **ef**（ef_search）  | 检索  | 搜索广度           | =K  | [K,10K]  | 召回↑延迟↑     |
|                    |     |                |     |          |            |

**关键**：M、efConstruction 建索引时定死（改要重建索引，贵）；ef 检索时动态调（便宜，日常主旋钮）。

**DB 类比**：M ≈ B-tree 扇出/分支因子；efConstruction ≈ 建索引认真程度；ef ≈ 查询扫描范围（像 LIMIT 但是搜索广度）。

**维度对 HNSW 的三重影响**：
1. **内存**：向量存储=维度×4B×向量数，图开销约 1.5-2 倍。100 万、M=30：1024 维≈10GB，3072 维≈30GB（翻 3 倍）。
2. **延迟**：距离计算 O(维度)，3072 维每次跳邻居 = 1024 维 3 倍计算，整体延迟线性涨。
3. **召回（最关键）**：维度灾难——高维空间稀疏，同 M 下邻居覆盖不足，召回掉。要保召回得加大 M 或 ef → 成本雪上加霜。

**反向验证 L05**：选 3072 不只存储翻 3 倍，还会逼大 HNSW 参数、延迟涨——真实代价比 L05 表格更大。1024 是甜点，HNSW 视角也成立（M=30 默认够用）。

**调参顺序**（口诀：先检索参数，再建索引参数，最后才动维度/切分）：
```
recall 不够 → ① 调 ef（最便宜）→ ② 调 efConstruction（重建）→ ③ 调 M（重建，影响最大）→ ④ 退回 L03/L05 审视切分/维度
```
评测驱动：50 题测试集跑 Recall@5，每次只动一个参数。recall 不够先调 ef 不是换高维——调参比换维度便宜得多。

**关联课程**：[L05 Embedding 选型](lessons/0005-embedding-selection.html)（维度权衡）· [L02 全流程地图](lessons/0002-rag-pipeline.html)（检索环节）

---

## 0008 · 混合检索的三种分数是什么？怎么判断混合检索"够了"？

**三种分数，来历不同**：

| 分数 | 怎么来 | 范围 | 有绝对意义吗 |
|---|---|---|---|
| **dense（cosine）** | query/doc 向量算夹角余弦 `A·B/(|A||B|)` | [-1,1] | ✅ 语义相似度，0.85 比 0.7 更像 |
| **BM25** | query 分词，每词算 TF×IDF×长度归一，求和 | [0,+∞) 无上界 | ❌ 相对分，只能同查询内比 |
| **RRF** | 只看排名 `Σ 1/(k+rank)`，多路求和 | (0, 路数/k] | ❌ 纯相对，数值本身没含义 |

**核心认知**：RRF 分数**没有绝对物理意义**——0.03278 不代表"32.78%相关"，纯粹是"多路排名共识分"，谁在多路都靠前谁分高。和 cosine（语义）、BM25（统计）本质不同。我们只 care RRF 的相对大小（排序），不解读数值。

**融合分数的意义**：唯一意义是**排序**——把两路召回并集按多路共识排个序，取 Top-K 喂 LLM。高 RRF = 两路都靠前 = 更可能相关 → 排前面。不解读数值。

**"够不够"看 Recall@K，不看分数**：RRF 只管排序，"够不够"要拿排序结果和 ground truth 比（捞全了没）。常见误区：以为 RRF 分高=检索好。错——RRF 分像 `ORDER BY` 排序键，Recall 像查全率，排序再漂亮不代表没漏数据。

**达标标准（可操作）**：
1. **Recall@5 ≥ 90%**（核心硬指标，Top-5 捞全 90% 相关文档；医疗/法律要 95%+，内部 FAQ 80% 也行，看业务对漏召回容忍度）
2. 标识符查询不再漏（ORA-01555/PGXC_STAT_ACTIVITY 能召回）
3. 语义查询不退步（不能为补标识符把语义类搞差）
4. 业务抽样 20 真实查询，Hit Rate@5 ≥ 95%

**何时停**：四条都满足 → 混合检索够了，进 L07 rerank 提 precision；有缺口 → 调各路 K / RRF k / 查切分 embedding。**别在 RRF 分数上纠结**——recall 达标就进 rerank，不达标是召回问题不是分数问题。

**口诀**：recall 达标进 rerank，不达标先调参；分数只管排，够不够看 Recall。

**关联课程**：[L06 混合检索](lessons/0006-hybrid-retrieval.html)（RRF 融合）· Q&A 0002（Recall@K 评估指标）

---

## 0009 · 双系统（经典）架构里是两个模型吗？数据怎么同步？

**核心澄清**：双系统的"双"指**两类检索引擎**（向量库 + 全文检索引擎），**不是两个 embedding 模型**。整个架构里只有**一个 embedding 模型**（做 dense）。

**数据流**：
```
chunks ─┬─→ 过 embedding 模型 → 向量 → 存【向量库 Milvus】（HNSW 索引）
        └─→ 原文 → 存【ES】（倒排索引，不跑模型）

query ─┬─→ 过 embedding 模型 → query 向量 → 查向量库 → dense Top-K
       └─→ 分词 → 查 ES → BM25 Top-K（TF×IDF 统计，无模型）
       → 应用层 RRF 融合
```

**关键**：BM25 **不是模型**，是 ES/OpenSearch 内置的统计算法，直接对原文建倒排索引，查询时统计词频——不跑任何 embedding。所以"两个模型分别抽取"是误解，只有一个模型（dense 向量化）。

**数据同步两份**：每个 chunk 既要向量化存向量库，又要原文存 ES，形态不同（向量 vs 原文倒排）。文档更新两个系统都要更新。这是双系统的代价。

**不能共用库**：向量库存不了倒排索引，ES 存不了向量相似度——两个系统数据形态不同，必须各存一份。

**vs 一站式（BGE-M3）**：
| | 双系统 | 一站式 |
|---|---|---|
| 模型 | 1 个（dense） | 1 个（BGE-M3 出 dense+sparse） |
| 系统 | 2 个（向量库+ES） | 1 个（Milvus） |
| BM25 来源 | ES 真 BM25（最稳） | BGE-M3 神经稀疏（罕见标识符弱） |
| 数据同步 | 两份 | 一份 |

**DB 类比**：双系统 ≈ 同一份数据在 MySQL 建表 + ES 建索引，各查再合并；一站式 ≈ 一个库支持两种查询。

**POC 路径**：先用 BGE-M3 一站式快速出 demo；规模大/标识符检索不达标 → 迁双系统（dense+ES BM25）。

**关联课程**：[L06 混合检索](lessons/0006-hybrid-retrieval.html)（双系统架构）· Q&A 0005（BGE-M3 稀疏 vs BM25）· Q&A 0006（BGE-M3 三合一用法）

---

## 0010 · 混合检索的升级路径：纯 dense → 一站式 → 双系统

**"上混合"是相对纯 dense 的升级，内部又分两档**：
```
纯 dense → 一站式（BGE-M3 dense+sparse）→ 双系统（dense + ES BM25）
            省事但稀疏弱                  最稳但维护两套系统
```
每步评测驱动，达标即停，别盲目升级。

**升级决策**：
1. 选 BGE-M3，先纯 dense 跑基线 Recall@5
2. recall 不够（标识符查询漏）→ 开 BGE-M3 一站式，评测 → 达标停
3. 一站式还不够（BGE-M3 sparse 对罕见标识符弱）→ 升级双系统（BGE-M3 dense + ES BM25），评测 → 达标停
4. 双系统还不够 → 问题不在检索架构，退回查切分/embedding/测试集

**易混点①：一站式是 BGE-M3 特有福利**。"一个模型出 dense+sparse"是 BGE-M3 的能力。选其他 dense 模型（GTE-Qwen2、bge-large-zh）没有自带 sparse，要混合只能直接上双系统，没有一站式这档。

**易混点②：双系统不是扔掉 BGE-M3**。双系统 = BGE-M3 dense + ES BM25，扔掉的是 BGE-M3 的 sparse（换成 ES 真 BM25），dense 那路照旧用 BGE-M3（它的 dense 很强）。即只换 sparse 那路，不换 dense 模型。

**核心原则**：逐步升级，每步评测，达标即停。别一上来双系统（过度工程），也别在纯 dense 上死磕（治不了标识符）。

**关联课程**：[L06 混合检索](lessons/0006-hybrid-retrieval.html)·[L05 Embedding 选型](lessons/0005-embedding-selection.html)（BGE-M3）· Q&A 0005/0006/0009

---

## 0011 · LLM-as-a-Judge 到底怎么当裁判？举个实际例子

**一句话**：LLM-as-a-Judge = 不用人评，让 LLM 按你给的评分标准，自动给答案打分。

**为什么需要**：人工评 RAG 答案慢（50 条要半天）、贵（1000 条好几个人天）、不一致（同一条张三 0.8 李四 0.6）。LLM-as-Judge 几秒一条、几分钟跑千条、成本几块钱、还一致。

**怎么当裁判——核心是"评分 prompt"**：给 LLM 一段 prompt 写清评分标准 / 怎么打分 / 输出格式，LLM 按它对每条样本打分。

**实际场景：评 DBA Copilot 的 Faithfulness（忠实度）**

用户问："ORA-01555 怎么处理？"

检索上下文（chunk）：
> ORA-01555 快照过旧错误，通常由长事务或 undo 表空间不足引起。处理：增大 UNDO_RETENTION 参数或扩展 undo 表空间。

LLM 答案：
> ORA-01555 是快照过旧错误，由长事务引起。处理方法是增大 UNDO_RETENTION 或扩展 undo 表空间，也可以重启数据库。

LLM-as-Judge 分两步（都是 LLM 干）：

第 1 步，拆答案成 claim（原子事实）：
1. ORA-01555 是快照过旧错误
2. 由长事务引起
3. 处理：增大 UNDO_RETENTION
4. 或扩展 undo 表空间
5. 也可以重启数据库

第 2 步，逐个判断能否被上下文支撑：
1.✓ 上下文有"快照过旧" 2.✓ 上下文有"长事务" 3.✓ 上下文有 4.✓ 上下文有 5.✗ 上下文没有"重启数据库"——编的

算分：Faithfulness = 4/5 = 0.8

**裁判 prompt 简化版**（RAGAS 实际更细）：
```
你是评估裁判。判断答案的每个事实陈述能否被上下文支撑。
【上下文】{retrieved_contexts}
【答案】{response}
步骤：1.拆成原子事实 2.逐个判断能否由上下文支撑 3.输出 能支撑数/总数
输出 JSON：{"claims":[...],"supported":4,"total":5,"score":0.8}
```
50 条评估集跑一遍 → 50 个分数 → 平均 = 系统的 Faithfulness 指标。

**三种用法**：
1. 单答案打分（score-based）：一个答案打 0-1 分，如 Faithfulness。最常见。
2. 对比评估（pairwise）：给两个答案 A/B 问哪个好。A/B 测试两模型或两版 prompt 用。
3. 拆解式（decomposed）：如 Faithfulness 拆 claim 逐个判断，比直接问"忠实吗"更准（大问题拆小判断，LLM 每步只做简单决策，出错率低）。

**为什么会偏差**（具体例子）：
- 偏好长答案（verbosity bias）：简短正确答案 A vs 啰嗦正确答案 B，LLM 可能给 B 更高分（"看着更详细"）
- 偏好自己风格（self-enhancement）：GPT-4 评 GPT-4 答案偏高
- 不稳定：同一份答案两次跑分数不同（温度不为 0 时）

→ 所以 L09 强调"LLM 评估不是金标准，人工抽检校准"。缓解：用更强模型评、固定 seed/低温、人工标注校准集。

**DBA 类比**：LLM-as-Judge = 让"自动化脚本 DBA"按你写的评分规则批改运维答卷。比人工快，但有自己的坏习惯（喜欢长答案、偏心自己人、心情不稳），关键考试要抽查复核。

**关联课程**：[L09 RAG 评估](lessons/0009-evaluation.html)· Q&A 0010

---

## 0012 · 评估用的 retrieved_contexts（检索到的 chunk）从哪来？系统能输出吗？

**能，而且是 RAG 系统本来就有的中间产物，不是评估时特殊造的。**

**两个角度都能"输出"：**

**①评估角度——从哪来**：RAG 链路里 retrieve 环节的输出就是 retrieved_contexts（Top-K chunk）。生成时：retrieve 拿到 chunks → 拼成 context → 喂 LLM 生成。chunks 是生成前的中间产物。评估时把同一份 chunks 截留下来，连同 response、reference 喂给 RAGAS。所以评估用的 context = 系统运行时的中间数据，记日志即可拿到。

**②产品角度——给用户看**：RAG 系统通常会把检索到的 chunk 作为"引用来源"展示给用户（L08 引用机制 [1][2]）。如 Dify 开"引用与来源"开关，答案下方显示参考来源 + 片段预览，点开看原文。不会展示全部 chunk 原文（太长），通常显示来源标题 + 片段。运维场景 DBA 看完 AI 诊断还要点原文确认，这个特别重要。

**工程上怎么拿**：
- LangChain：`retriever.invoke(query)` 返回 `List[Document]`，每个 Document 含 `page_content` + `metadata`，本来就能拿到
- LlamaIndex：查询返回的 response 里有 `source_nodes`，可取出检索到的 chunk
- Dify：聊天流开"引用与来源"开关，检索结果随答案返回前端
- 通用做法：日志记录每次查询的检索结果（chunk 内容 + 来源 + 分数），评估时从日志取

**为什么重要——评估可复现性**：要评估就得把"检索到什么"和"生成了什么"对应起来。RAG 系统设计时要记录中间产物（retrieved chunks / response / 用的 prompt），评估时才能复现、定位瓶颈。不记录中间产物的系统事后没法评估——所以 L09 说评估要从系统设计阶段就考虑，不是做完再补。

**DBA 类比**：retrieved_contexts = SQL 的执行计划。数据库执行 SQL 时本来就会生成执行计划（中间产物），EXPLAIN 能拿出来看；RAG 的检索 chunks 也是执行时的中间产物，可以"截留"出来评估或展示给用户。

**关联课程**：[L09 RAG 评估](lessons/0009-evaluation.html)·[L08 上下文组装与 Prompt 工程](lessons/0008-context-prompt.html)（引用机制）· Q&A 0011

---

## 0013 · L09 深入三问：RAGAS 怎么跑 / 评估集规模 / 多轮对话评估

### 一、RAGAS 实际怎么跑（代码）

最小可跑示例（RAGAS v0.4，能读懂即可）：

```python
from ragas.dataset_schema import SingleTurnSample
from ragas import EvaluationDataset, evaluate
from ragas.llms import llm_factory
from ragas.metrics import (
    LLMContextPrecisionWithReference,
    LLMContextRecall,
    Faithfulness,
    ResponseRelevancy,
)

# 1. 裁判 LLM（评估专用，和生成答案的 LLM 可以不同——建议裁判用更强模型）
evaluator_llm = llm_factory("gpt-4o-mini")

# 2. 评估集：每条 4 字段
samples = [
    SingleTurnSample(
        user_input="ORA-01555 怎么处理？",
        retrieved_contexts=["ORA-01555 快照过旧，由长事务或 undo 不足引起。增大 UNDO_RETENTION 或扩展 undo 表空间。"],
        response="ORA-01555 是快照过旧错误，增大 UNDO_RETENTION，也可以重启数据库。",
        reference="ORA-01555 快照过旧，增大 UNDO_RETENTION 或扩展 undo 表空间。",
    ),
    # ... 共 50 条
]
dataset = EvaluationDataset(samples)

# 3. 选四指标
metrics = [
    LLMContextPrecisionWithReference(llm=evaluator_llm),
    LLMContextRecall(llm=evaluator_llm),
    Faithfulness(llm=evaluator_llm),
    ResponseRelevancy(llm=evaluator_llm),
]

# 4. 跑评估
results = evaluate(dataset=dataset, metrics=metrics)
print(results)
# {'context_precision': 0.85, 'context_recall': 0.87, 'faithfulness': 0.80, 'response_relevancy': 0.91}
```

**关键点**：
- `evaluator_llm` 是裁判，和生成答案的 LLM 可不同。建议裁判用更强模型（GPT-4o 评 GPT-3.5 生成），降 self-enhancement 偏差。
- 每条样本 4 字段：`user_input` / `retrieved_contexts` / `response` / `reference`。reference 不是所有指标都需要（Faithfulness/ResponseRelevancy 不需要，Context Recall 需要），但建议都备齐。
- 上面这条样本的 `response` 含"也可以重启数据库"（编的），Faithfulness 会算出 0.8（4/5 claim 可支撑）——对应 Q&A 0011 的例子。
- 成本：50 条 × 4 指标，Faithfulness 要拆 claim 多次调 LLM，总计几百次调用，gpt-4o-mini 几块钱，gpt-4o 几十块。
- LangChain 集成：RAGAS 能从 LangChain RAG 链路的 trace 直接转评估集，原理就是上面把 retrieved_contexts / response 凑齐。

### 二、评估集规模怎么定（50 够不够）

**看阶段和目的**：
- POC 初期 30-50 条够——目的是快速看趋势、定位大瓶颈，不需要统计显著性。50 条能看出"Context Recall 0.7"这种明显问题。
- 正式评估 100-200 条——要给客户出报告、做 A/B 对比时，样本太少波动大，100+ 条指标更稳。
- 回归测试集 200-500 条——每次改代码/换模型跑回归，覆盖各类 case。

**怎么判断够不够——看指标波动**：
- 跑两次（或重采样），分数波动 < 0.03（3 个百分点）说明样本够稳。
- 波动大（如 0.65 vs 0.72）→ 样本太少，加样本。

**比规模更重要的是质量和分层**：
- 50 条高质量分层（简单/多步/边界/易混淆各覆盖）比 200 条全是简单题有用。
- 易错的边界 case（知识库没有的、易混淆的）宁可多标几条。

**增量策略**：先标 50 条跑基线 → 优化过程中把"踩坑的 case"（答错的、用户投诉的）持续加进评估集 → 滚动增长。评估集是活的，不是一次定死。

**结论**：50 条 POC 够，正式交付不够，持续滚动增长。DBA 类比：测试集不在行数多，在覆盖各种边界（NULL/空表/大表/并发）。

### 三、多轮对话怎么评估

运维场景多轮很常见：用户问"ORA-01555 怎么处理" → AI 答 → 用户追问"那 UNDO_RETENTION 设多少"（"那"指代上一轮）。

**难点**：单轮评估看"这一问这一答"，多轮要看"对话历史 + 当前问 → 当前答"。后续轮次依赖前文（指代消解、上下文继承），中间轮检索不同 chunk。

**RAGAS 对多轮支持**：
- 有 `MultiTurnSample` 封装多轮对话，但多轮 RAG 的检索/生成指标支持弱于单轮——多轮的"context"难定义（哪轮的？）。
- 多轮指标偏 agent 场景（TopicAdherence 话题 adherent、ToolCallAccuracy）。

**实操做法（运维 POC 推荐）**：
1. 拆成单轮评估：把多轮拆成多个 (问题, 上下文, 答案, 标准答案) 单轮样本，用单轮指标评。简单但丢了"上下文继承"评估。
2. 对话级指标：人工或 LLM 评整段对话的连贯性、上下文是否正确继承。
3. 关键检查点：指代消解（第 2 轮"它"指什么对不对）、上下文继承（第 1 轮信息第 2 轮有没有沿用）、话题漂移。
4. 查询重写单独评估：多轮 RAG 通常有"查询重写"步骤（把"那 UNDO_RETENTION 设多少"重写成"ORA-01555 的 UNDO_RETENTION 设多少"再检索），单独评估重写质量——这步是多轮 RAG 关键，重写错了后面全错。

**售前视角**：多轮评估自动化还不成熟，人工为主。POC 时准备 5-10 个多轮场景（每个 3-5 轮），人工评估连贯性 + 把关键轮拆出来单轮评估。

**DBA 类比**：多轮对话 = 存储过程/事务（有上下文依赖），单轮 = 单条 SQL。评估存储过程比单条 SQL 复杂，要考虑事务内状态变化。

**关联课程**：[L09 RAG 评估](lessons/0009-evaluation.html)· Q&A 0011（LLM-as-Judge）/ 0012（retrieved_contexts）

---

## 0014 · 多轮对话评估场景"具体怎么准备"？代码没法编写吗？

**关键澄清**：多轮场景是**人工设计**的（DBA 领域知识），不是代码生成的。"代码没法编写"场景是对的——这部分本来就不是写代码，是写剧本。代码只负责跑剧本、记结果。这也是为什么说"多轮评估自动化不成熟，人工为主"。

**分工**：
- 人（DBA）：设计多轮场景、标注每轮标准答案和考点、写成文件
- 代码：读场景文件 → 模拟多轮对话 → 收集系统实际回答 → 人工/LLM 打分

**准备 4 步**：

**第 1 步：从真实 DBA 排查流程抽象场景**。DBA 排查问题本来就是多轮的（现象→细节→处理→参数→验证），把这种真实流程抽象成评估场景，每个 3-5 轮。

**第 2 步：设计每个场景的多轮 + 标注**。以"慢SQL诊断多轮追问"为例：
- 轮1 "数据库最近变慢，怎么排查慢SQL？" → 考点：基础检索
- 轮2 "上面说的那个视图，具体叫什么？怎么查？" → 考点：指代消解+上下文继承（"那个视图"指轮1的 pg_stat_statements）
- 轮3 "那查出来慢SQL后，怎么分析执行计划？" → 考点：话题推进
- 轮4 "如果是索引缺失，怎么加索引不影响业务？" → 考点：条件分支追问

轮2的指代消解是关键——RAG 系统得先理解"那个视图"指什么，再查询重写成"pg_stat_statements 怎么查"去检索。这步错后面全错。

**第 3 步：存成结构化文件**（JSON），含 scenario_id / background / turns（每轮 user_input / reference_answer / test_point）。

**第 4 步：代码跑剧本、记结果**：
```python
import json
scenarios = json.load(open("multi_turn_eval.json"))
results = []
for s in scenarios:
    chat_history = []  # 维护对话历史（多轮关键）
    for turn in s["turns"]:
        answer = my_rag_system.chat(turn["user_input"], history=chat_history)
        results.append({..., "actual_answer": answer, "reference_answer": turn["reference_answer"], "test_point": turn["test_point"]})
        chat_history.append({"user": turn["user_input"], "assistant": answer})
# 人工或 LLM 对比 actual vs reference 打分，重点看 test_point="指代消解" 的轮次
```
代码只干：读场景 → 喂系统（带历史）→ 记实际回答。打分仍要人工或 LLM。chat_history 维护对话历史是多轮关键——系统要能用历史做查询重写+上下文继承。

**5-10 个场景怎么凑**：按运维排查类型分，每类 1-2 个：
- 慢SQL诊断（现象→视图→执行计划→加索引，考指代+话题推进）
- 会话/锁分析（查会话→查锁→杀会话，考上下文继承）
- 参数调优（哪个→设多少→怎么验，考细化追问）
- 故障处理（现象→根因→处理→验证，考完整排查链）
- 边界（知识库没有的，考防幻觉）
- 话题切换（聊A→换B→回A，考上下文管理）
7 类 × 1-2 个 = 5-10 个，覆盖够。

**DBA 类比**：多轮评估场景 = DBA 的"故障排查 SOP 剧本"。把真实排查的多步追问写成剧本，让 AI 按剧本演，看每步答得对不对。代码是"演员和录像机"（按剧本演、记下来），剧本是 DBA 写的——所以代码没法编写场景，是对的。

**关联课程**：[L09 RAG 评估](lessons/0009-evaluation.html)· Q&A 0013（多轮对话评估总览）

---

## 0015 · 阿里云的 Dify RAG 方案能用吗？是不是整好了？

**结论**：能用，且和用户（阿里云售前背景）很对口。但它"整好"的是 Dify 部署 + 电商客服 RAG 教程骨架，不是运维 DBA Copilot 成品——料和 prompt 要换。

**这个方案是什么**：阿里云"技术解决方案"之一——"搭建基于RAG的AI电商客服助手（在线部署）"。用 Dify + RAG 在阿里云上搭 AI 电商客服，含在线部署 + 手把手教程。keywords：智能问答/Dify/电商客服助手/检索增强生成。

**整好了 / 没整好**：
- 整好：①Dify 在阿里云上的在线部署 ②一套完整 RAG 客服 demo 流程教程 ③模型/知识库/应用基础配置示范
- 没整好：①场景是电商客服不是运维知识库，要换料 ②运维文档要替换进去 ③DBA 诊断 prompt 要换（L08 那套）④检索调参（分段/TopK/rerank）按运维语料重调（L10 步骤2-3）

**关键**：技术栈（Dify+RAG）和运维场景完全一样，流程能照搬，但料和 prompt 要换成运维的。不能直接拿来当 DBA Copilot 用。

**分场景建议**：
- 学 L10 / 最快出 demo → cloud.dify.ai（免费、不部署、注册即用）。阿里云方案要在线部署+买资源+跑教程，学概念阶段绕远。
- 想练阿里云部署 + 看官方 RAG 教程 → 用这个方案，把电商客服文档换成运维文档、prompt 换成 DBA 诊断，技术流程照搬。
- L11 企业 POC / 给阿里云客户演示私有化 → 这个方案最合适（私有化+合规+国内快+能讲"阿里云方案"）。

**注意**：页面是前端渲染（SPA），正文抓不全，上述基于 meta（title/keywords/description）+ 阿里云技术解决方案一般模式判断。点进页面后建议确认部署后给的是"Dify 访问地址+教程"还是更完整的成品。

**关联课程**：[L10 Dify 搭运维知识库 RAG demo](lessons/0010-dify-demo.html)· Q&A 0013

---

## 0016 · 华为云 CSS（OpenSearch）能做向量库吗？只有 CSS 会语义检索打折吗？

**能。CSS/OpenSearch 本身支持向量检索（kNN），不只是 BM25 稀疏。** "没有向量库只有 CSS" 多半是误解——CSS 自己就是向量库。

**CSS 向量能力**（华为云 CSS 基于 OpenSearch/ES）：
- knn_vector 字段类型 + HNSW 算法（近似 kNN）+ 暴力扫描（精确 kNN）
- 支持余弦/欧氏/内积距离
- nmslib/faiss 引擎
- 支持向量+标量过滤的混合查询
- Dify 本身支持 OpenSearch 作为向量存储后端

**所以 CSS 既能稀疏（BM25）又能稠密（kNN），一个引擎两路。** 架构图只画 CSS 一个框，是因为一个引擎同时承担稠密+稀疏，不是缺向量库。

**但要确认方案是否启用了 kNN**（两种情况天差地别）：
- 启用 kNN → CSS = 稠密+稀疏混合检索（L06），语义检索不打折，一个引擎两路省一个组件
- 只用 BM25 没开 kNN → 确实只有稀疏，语义检索打折（搜"数据库慢"匹配不到"性能下降"）

**怎么确认**：①看 CSS 索引 mapping 有没有 knn_vector 字段 ②Dify 知识库检索设置能不能选"向量检索/混合检索" ③看方案文档检索方式。mapping 有 knn_vector 就是开了向量。

**DBA 类比**：OpenSearch/CSS 像 PostgreSQL——既能 B-Tree（精确/BM25）又能 pgvector（向量 kNN），一个引擎两种索引，不必分开两个库。不会因"PG 有 B-Tree"就说它不能存向量，CSS 同理。

**售前视角**：CSS 做混合检索（稠密+稀疏）反而有优势——一个引擎两路，省一个向量库组件，运维简单（对比 Q&A 0009 双系统要 dense+ES 两套）。但要确认真的开了 kNN，别被"只有 CSS"误导以为没稠密。

**关联课程**：[L05 Embedding 选型](lessons/0005-embedding-selection.html)·[L06 混合检索](lessons/0006-hybrid-retrieval.html)· Q&A 0009（双系统）/ 0010（升级路径）

---

## 0017 · 华为云 Dify 方案架构确认：CSS 语义搜索 + bge-m3 + reranker（完整混合检索）

**确认**：华为云"快速搭建 Dify-LLM 应用开发平台"方案（v3.0.0，2026.2 更新）的"知识库搜索增强版"开了 CSS 语义搜索（向量），不是只有稀疏 BM25。检索架构完整，语义检索不打折。

**检索架构（对应 L05-L07）**：
- Embedding：bge-m3（L05 推荐的运维首选），单独一台 FlexusX 部署
- 向量存储+检索：CSS OpenSearch，"提供在线分布式搜索及语义搜索等功能"——语义搜索 = kNN 向量检索（稠密），同时 CSS 也能 BM25（稀疏）= 一个引擎两路混合（L06）
- Reranker：bge-reranker-v2-m3（L07 推荐的运维首选），和 embedding 同机部署
- 即 bge-m3 → CSS(kNN稠密+BM25稀疏) → bge-reranker-v2-m3，完整 L06 混合 + L07 rerank

**三个版本选哪个**：
- 社区版单机（2-5元）：只 FlexusX 一台，没 CSS 没模型机，知识库用 Dify 内置存储——能力弱，不适合 RAG demo
- **知识库搜索增强版（6-12元）**：有 CSS 语义搜索 + bge-m3 + reranker——**做运维 RAG demo 选这个**
- CCE 高可用（35-70元）：全量（CCE+Redis+RDS PG+CSS+ELB+NAT+OBS）——生产级

**关键**：用户之前担心"没有向量库只有 CSS"——若看的是单机版确实没 CSS（Dify 用内置存储）；但"知识库增强版"有 CSS 语义搜索（向量），配齐 embedding+reranker，正是 L05-L07 推荐架构。印证 Q&A 0016：CSS 能做向量库，华为云方案用了。

**售前点评**：架构配得正——bge-m3 + bge-reranker-v2-m3 是运维首选，CSS 做 kNN+BM25 一个引擎两路比双系统省事（Q&A 0009）。私有化部署到华为云，合规+国内快。成本上"知识库增强版"6-12元体验，做 demo 够用。

**关联课程**：[L05 Embedding 选型](lessons/0005-embedding-selection.html)·[L06 混合检索](lessons/0006-hybrid-retrieval.html)·[L07 Rerank](lessons/0007-rerank.html)·[L10 Dify demo](lessons/0010-dify-demo.html)· Q&A 0016（CSS 能力）

---
