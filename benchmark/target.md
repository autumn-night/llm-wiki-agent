# LLM-Wiki Benchmark 规划

## 1. 目标

对 llm-wiki 的 `ingest`、`lint`、`query` 三大环节进行**量化测速**（时间 + Token），并**固定 query 输出格式**使其可直接被 `ragas` 评估，用于后续与 RAG+知识图谱方案做横向对比。

---

## 2. 环境前提

| 依赖 | 状态 | 备注 |
|------|------|------|
| Python 3.13 | ✅ | 满足 `>=3.10,<3.14` |
| `litellm` | ❌ | `pip install litellm` |
| `networkx` | ❌ | `pip install networkx` |
| `ragas` | ❌ | `pip install ragas` |
| LLM API Key | ❌ | 需配置 `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` 或 kimi proxy |
| PDF 源文件 | ⚠️ | 位于 `examples/`（24 篇），需 `pdf2md` 转 `raw/` |

> **准入检查**：运行 `python tools/health.py` 零报错后再开始测速。

---

## 3. 测速指标体系

### 3.1 通用指标（所有环节）

| 指标 | 符号 | 采集方式 |
|------|------|---------|
| 总耗时 | `T_total` | `time.perf_counter()` 包裹入口函数 |
| Prompt Tokens | `tok_in` | `response.usage.prompt_tokens` |
| Completion Tokens | `tok_out` | `response.usage.completion_tokens` |
| Total Tokens | `tok_total` | `tok_in + tok_out` |
| LLM RTT | `T_llm` | 单次 `completion()` 耗时 |

### 3.2 分环节指标

#### Ingest
- `T_pdf2md`：PDF 转 Markdown（本地计算，独立计时）
- `T_context_build`：读取 index/overview/recent sources
- `T_ingest_llm`：单次 LLM 调用（source + entity + concept + overview 合成）
- `T_write`：写入 wiki 文件
- `T_validate`：后验校验

#### Lint
- `T_lint_total`：完整流程
- `T_structural`：孤儿页 / 断链 / 缺失实体（纯代码，零 LLM）
- `T_llm_contradictions`：矛盾检测
- `T_llm_gaps`：数据缺口分析

#### Query
- `T_retrieval`：索引匹配 + 可选 graph 扩展
- `T_llm_answer`：答案合成
- `retrieved_pages`：召回页面数
- `retrieved_chars`：召回上下文总字符数

---

## 4. Query 输出标准化（ragas 兼容）

ragas 要求每条样本包含：`question`, `answer`, `contexts`, `ground_truth`。llm-wiki 的 query 流程必须改造为**可复现、可结构化输出**。

### 4.1 输出格式规范

`benchmark/results/ragas_eval.jsonl`，每行一条样本：

```json
{
  "question": "What are the main attack vectors in software supply chains?",
  "answer": "...",
  "contexts": [
    "page: sources/ladisa-2023-sok.md\n---\nContent...",
    "page: concepts/SupplyChainAttack.md\n---\nContent..."
  ],
  "ground_truth": "The main attack vectors include dependency confusion, typosquatting, malicious code injection, and compromised build pipelines.",
  "metadata": {
    "retrieved_pages": 5,
    "retrieved_chars": 12400,
    "T_retrieval_ms": 12.3,
    "T_llm_ms": 4200,
    "tok_in": 3800,
    "tok_out": 420
  }
}
```

### 4.2 Contexts 提取规则

Query 召回的页面内容直接作为 `contexts` 列表元素。为便于 ragas 溯源，每条 context 前缀固定格式：

```
page: <relative-path>.md
---
<原始页面内容>
```

若启用 graph 扩展，被扩展召回的邻居页面也需加入 `contexts`。

### 4.3 Answer 格式规范

要求 LLM 输出固定结构（通过 system prompt 强制）：

```markdown
## Answer
<正文，必须包含 [[PageName]] 引用>

## Sources
- [[PageName]] — 引用理由
```

Benchmark 采集 `## Answer` 与 `## Sources` 之间的内容作为 `answer` 字段传入 ragas。

### 4.4 Ground Truth 生成

由于当前无人工标注，采用**自动化 ground truth 生成**：

1. **全文基线法**：将 24 篇 PDF 的 Markdown 全文拼接，直接用 LLM 回答测试问题（不经过 wiki 检索），其输出作为 `ground_truth`。
2. **缓存机制**：ground truth 只生成一次，写入 `benchmark/ground_truth.json`，后续复用。

---

## 5. 测速代码架构

```
benchmark/
├── target.md              # 本文件
├── requirements.txt       # benchmark 专用依赖（ragas, pandas, matplotlib）
├── common.py              # MetricsCollector + Timer + LLM monkey-patch
├── run_ingest.py          # 全量 ingest 测速
├── run_lint.py            # lint 多轮测速
├── run_query.py           # query 标准化测速 + ragas 数据产出
├── generate_ground_truth.py  # 基于全文生成 ground truth
├── report.py              # 汇总所有 jsonl → summary.md
└── results/
    ├── ingest.jsonl
    ├── lint.jsonl
    ├── query.jsonl
    ├── ragas_eval.jsonl   # 直接喂给 ragas 的数据
    └── summary.md
```

### 5.1 common.py 核心设计

```python
# 伪代码
class MetricsCollector:
    def record(self, op: str, t_llm: float, tok_in: int, tok_out: int, model: str)
    def to_jsonl(path)
    def summary() -> dict

@contextmanager
def timer(name: str, store: dict):
    t0 = perf_counter(); yield; store[name] = perf_counter() - t0

def patch_litellm():
    # Monkey-patch tools.ingest/lint/query 的 call_llm，注入计时与 token 采集
    # 返回原函数引用，便于 unpatch
```

### 5.2 run_query.py 核心流程

```python
for q in TEST_QUESTIONS:
    with timer("retrieval", meta):
        pages = find_relevant_pages(q, index)
    contexts = [format_context(p) for p in pages]
    
    with timer("generation", meta):
        answer = call_llm(prompt_with_schema(q, contexts))
    
    record = {
        "question": q,
        "answer": extract_answer(answer),
        "contexts": contexts,
        "ground_truth": GROUND_TRUTH[q],
        "metadata": {**meta, **collector.last_call()}
    }
    append_jsonl("results/query.jsonl", record)
```

### 5.3 ragas 评估入口

```python
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall

ds = Dataset.from_json("benchmark/results/ragas_eval.jsonl")
results = evaluate(ds, metrics=[...])
results.to_pandas().to_markdown("benchmark/results/ragas_score.md")
```

---

## 6. 测试问题集（Test Suite）

共 8 题，覆盖事实检索、综合推理、跨文档对比三类：

| ID | 问题 | 类型 |
|----|------|------|
| Q1 | What are the main attack vectors in software supply chains discussed across all sources? | 综合 |
| Q2 | How does PyPI ecosystem security compare to NPM according to the wiki? | 对比 |
| Q3 | Summarize the detection techniques for malicious packages in open-source ecosystems. | 综合 |
| Q4 | What role do LLMs play in supply chain security based on the ingested papers? | 事实 |
| Q5 | List the key frameworks, tools, or datasets mentioned for analyzing software supply chains. | 事实 |
| Q6 | What contradictions exist between different papers regarding the effectiveness of static analysis? | 推理 |
| Q7 | Which papers discuss dependency confusion or dependency chaos, and what are their conclusions? | 事实 |
| Q8 | How is the concept of SBOM used in risk assessment? | 事实 |

---

## 7. 执行步骤

1. **环境准备**
   - `pip install litellm networkx pymupdf4llm ragas`
   - 配置 LLM API Key，验证连通性
   - 运行 `health` 确认基线通过

2. **数据准备**
   - `python tools/pdf2md.py examples/*.pdf` → `raw/`
   - `python benchmark/generate_ground_truth.py` → `benchmark/ground_truth.json`

3. **全量 Ingest**
   - `python benchmark/run_ingest.py raw/*.md`
   - 产出 `benchmark/results/ingest.jsonl`

4. **Lint 测速**
   - `python benchmark/run_lint.py --runs 3`
   - 产出 `benchmark/results/lint.jsonl`

5. **Query 测速 + ragas 数据生成**
   - `python benchmark/run_query.py`
   - 产出 `benchmark/results/query.jsonl` + `benchmark/results/ragas_eval.jsonl`

6. **ragas 评估**
   - `python benchmark/run_ragas.py`
   - 产出 `benchmark/results/ragas_score.md`

7. **汇总报告**
   - `python benchmark/report.py`
   - 产出 `benchmark/results/summary.md`

---

## 8. 关键约束

- **不修改 `tools/` 源码**：通过 `common.py` 的 monkey-patch 注入计量逻辑，保持工具链纯净。
- **ingest 串行**：共享状态文件（index.md / overview.md）禁止并发。
- **ground truth 一次性生成**：避免 LLM 随机性污染评估基线。
- **context 必须可溯源**：每条 context 必须带 `page: <path>` 头，方便人工抽查与 ragas 调试。
