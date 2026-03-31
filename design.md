# Agent-Oriented Semantic Search MVP Design

## 0. 文档导航

- 架构图版（评审）：[design-architecture.md](./design-architecture.md)
- 开发执行版（任务清单）：[design-tasks.md](./design-tasks.md)

本文件保留完整系统设计信息，作为主文档。

## 1. 背景与目标

在 Agent 时代，搜索系统需要直接服务任务型查询，而不仅是关键词匹配。本文档定义一个最小可用（MVP）的语义搜索系统：输入用户 query，返回最相关的 top(k) 段落级证据（支持网页与 PDF），作为 Agent 后续推理和回答的高质量上下文。

目标：
- 提供稳定的 `POST /v1/search` 检索 API。
- 支持 `5 路 query 重写 + 混合召回 + 外部 reranker`。
- 在小规模垂直语料上快速验证效果与可用性。
- 技术路线面向未来超大规模数据（百亿网页、50 亿 PDF 页）演进。

## 2. 范围与非目标

MVP 范围：
- 数据类型：网页、PDF 页面。
- 检索粒度：chunk（段落级证据）。
- 召回：实体倒排（ES）+ 向量召回（Milvus）。
- 排序：外部 reranker。
- 更新策略：日级离线更新。

非目标（后续阶段）：
- 端到端答案生成（本期只做检索与排序）。
- 近实时索引更新（分钟级）。
- 全网规模直接上线。

## 3. 总体架构

核心组件：
- API Service（Go）：对外搜索接口与查询编排。
- Query Rewrite Service（外部 API）：将原 query 重写为 5 路。
- Entity Inverted Index（ES）：实体/术语到 chunk_id 的倒排映射。
- Vector Store（Milvus）：chunk 向量检索。
- Metadata Store（关系型或 KV）：chunk 文本与来源元信息。
- Reranker（外部 API）：对候选集合统一精排。
- Ingestion Pipeline（Go 离线任务）：解析、分块、实体抽取、索引写入。

## 4. 数据模型与索引设计

### 4.1 主键规范
- `doc_id`：原始文档唯一标识（URL 规范化哈希 / PDF 文档 ID）。
- `chunk_id`：全局唯一，建议由 `doc_id + page_no + chunk_no` 生成稳定哈希。
- 约束：ES、Milvus、Metadata Store 均使用同一 `chunk_id`。

### 4.2 ES（实体倒排）
索引建议：`entity_postings_v1`

关键字段：
- `entity_key`（keyword）：归一化实体/术语。
- `chunk_id`（keyword）
- `doc_id`（keyword）
- `source_type`（keyword: web/pdf）
- `lang`（keyword）
- `ts`（date）

说明：
- 只承担实体到 chunk 的高效候选召回，不依赖全文 BM25 主召回。

### 4.3 Milvus（向量索引）
Collection 建议：`chunk_vectors_v1`

字段：
- `chunk_id`（主键）
- `embedding`（vector<float>）
- `source_type`（scalar）
- `lang`（scalar）
- `ts`（scalar）

### 4.4 Metadata Store
建议表：`chunk_metadata`

字段：
- `chunk_id`（PK）
- `doc_id`
- `source_type`
- `title`
- `url`（web）
- `pdf_page`（pdf）
- `chunk_text`
- `token_count`
- `lang`
- `ingest_time`

## 5. 查询主链路（5 路重写）

### 5.1 流程
1. 接收 `query`。
2. 调用 rewrite 模型，生成 5 路子查询（q1..q5）。
3. 5 路并行执行混合召回：
   - 实体倒排召回（ES）
   - 向量召回（Milvus）
4. 每路输出最多 50 条候选 chunk。
5. 合并 5 路候选（上限 250），按当前策略先凑满再去重。
6. 调用外部 reranker 对候选精排。
7. 返回 top_k 段落证据。

### 5.2 重写类型建议
- 实体显式化
- 同义改写
- 背景扩展
- 约束强化
- 原 query 保真版

### 5.3 超时与降级
- 总超时预算：目标 P95 <= 800ms。
- 慢路降级：子路超时可跳过，不阻塞整体返回。
- rewrite 失败降级：使用原 query 单路召回 + rerank。

## 6. API 设计

接口：`POST /v1/search`

请求体（MVP）：
- `query` (string)
- `top_k` (int, default 10)
- `source_types` (array: web/pdf)
- `filters` (object)
- `request_id` (string)

响应体（MVP）：
- `hits[]`：
  - `chunk_id`
  - `snippet`
  - `score`
  - `source_type`
  - `url_or_doc_id`
  - `pdf_page`（可空）
  - `title`
- `debug`（可选）：
  - rewrite 列表
  - 每路召回数量
  - 合并后候选数量

## 7. 数据导入与更新（Ingestion）

离线日更流程：
1. 读取源数据（web/pdf）。
2. 文本抽取与清洗。
3. chunk 切分（含 overlap）。
4. 实体/术语抽取与归一化。
5. embedding 生成（外部 API）。
6. 同步写入 ES、Milvus、Metadata Store。

要求：
- 幂等导入（重复执行不产生不一致）。
- 失败重试与坏样本隔离。
- 导入任务输出统计报表（成功数、失败数、耗时）。

## 8. 数据清空策略

清空工具支持：
- 维度：`es | milvus | meta | all`
- 范围：按 dataset / tenant / 时间段
- 模式：`dry-run` 默认启用，必须 `--confirm` 才执行

原则：
- 分批删除，避免长时间锁与服务抖动。
- 操作日志可审计。
- 清空后可做一致性校验。

## 9. 评测方案（重点）

### 9.1 对比组
- Baseline-1：原 query + Milvus + reranker
- Baseline-2：原 query + ES 实体倒排 + Milvus + reranker
- Target：5 路 rewrite + ES 实体倒排 + Milvus + reranker

### 9.2 公开数据集
- BEIR：通用检索基准
- LoTTE：长尾与复杂查询
- MIRACL（或 Mr.TyDi）：多语言补充

### 9.3 指标
相关性：
- `Recall@50`, `Recall@100`
- `MRR@10`
- `nDCG@10`
- `Hit@1/3/10`

系统性：
- `P50/P95/P99 latency`
- `timeout_rate`
- `empty_result_rate`
- `rewrite_fail_rate`

### 9.4 分桶评测
按 query 类型分桶输出指标：
- 实体明确型
- 实体模糊型
- 长问句
- 短关键词
- 时间敏感型

### 9.5 在线 A/B（小流量）
- A：不开启 5 路 rewrite
- B：开启 5 路 rewrite

核心观察：
- Top1 CTR / Top3 CTR
- 停留时长
- 重搜率
- 守护指标（延迟、错误率、超时率）

## 10. SLO 与可观测性

初始 SLO：
- 查询链路 P95 <= 800ms
- API 错误率 < 1%
- 空结果率受控并持续监控

可观测性：
- request_id 贯穿各阶段日志
- 分阶段打点：rewrite、ES 召回、Milvus 召回、merge、rerank
- 指标面板：延迟、召回量、rerank 输入规模、降级触发率

## 11. Go 工程目录

```text
ai-search-v1/
  cmd/
    api/
    importer/
    cleaner/
    evaluator/
  internal/
    app/
    api/{handler,dto,middleware}
    query/{rewrite,entity,recall,rerank,pipeline}
    ingest/{source,parse,chunk,enrich,indexer}
    storage/{es,milvus,meta}
    eval/{dataset,runner,metrics,report}
    clean/{plan,execute}
    model/{embedding,rewrite,rerank}
    config/
    observability/
  pkg/{types,util}
  configs/
  deployments/{docker,k8s}
  scripts/
  test/{integration,e2e}
```

## 12. 里程碑与演进

M1（MVP 可用）：
- 单域小语料打通完整链路
- 完成离线评测对比与第一版线上 API

M2（稳定优化）：
- 重写策略优化与候选配额调参
- 加强缓存、并发与降级策略

M3（规模化准备）：
- 索引分片、冷热分层
- 多集群与异步流水线优化

## 13. 主要风险与缓解

风险：实体抽取漏召回
- 缓解：保留 Milvus 并行召回兜底

风险：5 路并发导致延迟抖动
- 缓解：超时预算、慢路跳过、候选截断

风险：候选重复导致 rerank 有效密度下降
- 缓解：监控重复率；若效果不达标，切换为先去重再补齐策略

风险：公开数据集与业务场景偏差
- 缓解：MVP 后补充业务小样本标注集做校准

## 14. 模型选型（MVP 固定）

为确保评测可复现与实现一致性，MVP 阶段固定使用以下模型：

- Embedding 模型：`Qwen3-Embedding-0.6B`
- Reranker 模型：`Qwen3-Reranker-0.6B`

约束：
- 在同一轮离线评测和线上 A/B 中，不混用其他 embedding 或 reranker 模型。
- 若后续替换模型，必须记录新模型版本并与当前基线做对比评测（`Recall@k`, `MRR@10`, `nDCG@10`, `P95`）。
