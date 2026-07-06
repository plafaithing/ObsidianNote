# RAG 课程 — 资源清单

> 知识从高质量资源获取，不依赖参数化记忆。课中引用此处的来源。
> 资源按用途分组，标注「主源」的为对应课程的主要取材对象。

## 一、面试聚焦（高频问题）
- [面试官狂问的 28 个 RAG 问题全解析](https://juejin.cn/post/7641142853817974835) — 掘金，从基础到架构优化。**L12 主源**
- [2025 最新整理 75 道高频 AI 大模型面试真题](https://zhuanlan.zhihu.com/p/1933262215072449236) — 知乎，含 RAG 部分
- [llm_interview_note](https://github.com/wdndev/llm_interview_note) — GitHub，含「实现一个简单的 RAG 系统」

## 二、企业实战（落地痛点）
- [10 万文档 RAG 落地实战：从 Demo 到生产，我踩过的所有坑](https://developer.aliyun.com/article/1707263) — 阿里云，检索慢/召回率低/部署复杂三大痛点。**L11 主源**
- [企业级 RAG 系统实战（2 万+文档）：10 个项目踩过的坑](https://www.53ai.com/news/RAG/2025101180154.html) — 53AI，附代码
- [企业大模型落地的现实解法：为什么 RAG 是绕不开的技术路径？](https://www.woshipm.com/ai/6235363.html) — 人人都是产品经理，售前视角。**L01/L11 引用**
- [企业 AI 落地九大问题](https://m.36kr.com/p/3734716520415236) — 36 氪

## 三、进阶技术（分块/检索/重排/评估）
- [RAG 进阶：Chunking、召回、Hybrid Search 与 Rerank](https://www.zata.cc/p/rag%E8%BF%9B%E9%98%B6chunking%E5%8F%AC%E5%9B%9Ehybrid-search-%E4%B8%8E-rerank/) — Zata，进阶全景
- [深入 RAG 架构：分块策略、混合检索与重排序的工程实现](https://zhuanlan.zhihu.com/p/2008281994912609286) — 知乎，有实测数据（rerank 让前 3 命中率 68%→89%）。**L07 引用**
- [RAG 工程全景](https://quant67.com/post/llm-infra/17-rag-engineering/17-rag-engineering.html) — quant67，工程视角
- [20 个 RAG 核心概念揭秘](https://gitcode.csdn.net/6a277cc910ee7a33f279963f.html) — GitCode
- [RAG 检索增强生成 wiki](https://wiki.hugogu.cn/zh/ai/tech/rag) — HugoGu，含切分策略对比表。**L01/L03 主源**
- [RAG 优化](https://github.com/Snailclimb/JavaGuide/blob/main/docs/ai/rag/rag-optimization.md) — JavaGuide，系统工程视角

## 四、分块策略（专题）
- [检索增强生成的分块策略指南](https://zilliz.com.cn/blog/guide-to-chunking-sreategies-for-rag) — Zilliz（Milvus 官方）。**L03 主源**
- [寻找最优文本分块策略](https://developer.nvidia.cn/blog/finding-the-best-chunking-strategy-for-accurate-ai-responses/) — NVIDIA，页面级分块评估

## 五、Rerank（专题）
- [RAG 超越语义搜索：如何用 Rerank 实现检索精度大幅提升](https://developer.cloud.tencent.com/article/2588231) — 腾讯云。**L07 主源**
- [什么是 RAG 中的 Rerank？](https://developer.huawei.com/consumer/cn/blog/topic/03207049130129199) — 华为开发者联盟

## 六、官方/权威教程（系统学习）
- 吴恩达 DeepLearning.AI 短课：Prompt Engineering / RAG / Agent（简历已列）
- [LangChain 官方文档](https://python.langchain.com/)
- [LlamaIndex 文档](https://docs.llamaindex.ai/)
- [Dify 官方文档](https://docs.dify.ai/)（用户已用过，低代码上手）
- [Milvus 官方教程](https://milvus.io/docs) / [Zilliz](https://zilliz.com.cn)
- [Ollama](https://ollama.com/) / [vLLM](https://docs.vllm.ai/) / [HuggingFace](https://huggingface.co/)

## 七、评估
- [RAGAS 官方文档](https://docs.ragas.io/) — RAG 评估框架。**L09 主源**
