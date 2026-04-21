# LLM-Wiki 构建与测速规划文档

## 1. 本地环境可行性评估

### 1.1 当前环境快照

| 组件 | 状态 | 说明 |
|------|------|------|
| Python | ✅ 3.13.12 | 满足 `>=3.10,<3.14` 要求 |
| `kimi-cli` | ✅ 已安装 | `/home/fct_traffic/.local/bin/kimi` |
| `litellm` | ❌ 未安装 | `requirements.txt` 声明但环境中缺失 |
| `networkx` | ❌ 未安装 | `requirements.txt` 声明但环境中缺失 |
| LLM API Key | ❌ 未配置 | 无 `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` 等环境变量 |
| `raw/` 目录 | ⚠️ 为空 | 仅含 `.gitkeep`，源文件实际位于 `examples/`（24 篇 PDF） |

### 1.2 关键依赖分析

- **ingest / lint / query** 均通过 `tools/*.py` 中的 `call_llm()` 函数调用 LLM，底层统一使用 `litellm.completion()`。
- **pdf2md** 支持四种后端：`auto`（默认）、`arxiv2md`、`marker`、`pymupdf4llm`。`auto` 对本地 PDF 会优先尝试 `marker`，失败回退 `pymupdf4llm`。
- **health** 为纯结构性检查，零 LLM 调用，无需额外依赖即可运行。

### 1.3 可行性结论

> **本地构建可行，但需完成以下前置准备：**

1. **安装 Python 依赖**：`pip install litellm networkx pymupdf4llm`（若使用 marker 还需额外安装 `marker-pdf`）。
2. **配置 LLM API**：设置环境变量（如 `OPENAI_API_KEY`、`ANTHROPIC_API_KEY` 或 kimi-cli 兼容的 proxy 端点）。`ingest.py` 默认读取 `LLM_MODEL` 环境变量（默认 `claude-3-5-sonnet-latest`）；`lint.py` 与 `query.py` 分别读取 `LINT_MODEL` / `QUERY_MODEL`。
3. **迁移 PDF 源文件**：将 `examples/` 下的 24 篇 PDF 通过 `pdf2md.py` 转换为 `raw/` 下的 Markdown，再执行 ingest。

---

## 2. 测速目标与指标定义

### 2.1 测速范围

本次 benchmark 覆盖 **ingest → lint → query** 三大核心环节，并对每个环节内部的关键子操作进行拆解计时与 token 计量。

### 2.2 指标矩阵

| 指标维度 | 符号 | 单位 | 采集方式 |
|---------|------|------|---------|
| **总耗时** | `T_total` | 秒 (s) | `time.perf_counter()` 包裹整个操作 |
| **PDF→MD 耗时** | `T_pdf2md` | 秒 (s) | 单独计时 `pdf2md.py` 调用 |
| **文件 IO 耗时** | `T_io` | 秒 (s) | 读取源文件 + 写入 wiki 页面的累计时间 |
| **LLM 调用耗时** | `T_llm` | 秒 (s) | `completion()` 请求往返时间（RTT） |
| **Prompt Tokens** | `Tok_prompt` | tokens | 从 `response.usage.prompt_tokens` 提取 |
| **Completion Tokens** | `Tok_completion` | tokens | 从 `response.usage.completion_tokens` 提取 |
| **Total Tokens** | `Tok_total` | tokens | `prompt + completion` |
| **Token 吞吐量** | `R_token` | tokens/s | `Tok_total / T_llm` |
| **单页成本系数** | `C_page` | tokens/页 | 用于横向对比不同长度文档 |

### 2.3 操作级测速对象

#### Ingest 环节
- `ingest_total`：处理单篇源文档的端到端时间
- `pdf2md`：PDF 解析与 Markdown 转换
- `read_source`：读取源文件（Markdown）
- `build_context`：组装 wiki context（index + overview + recent sources）
- `llm_source_summary`：生成 Source Page（summary, claims, quotes, connections, contradictions）
- `llm_entity_extraction`：识别并生成/更新 Entity Pages
- `llm_concept_extraction`：识别并生成/更新 Concept Pages
- `llm_overview_update`：更新 overview.md（可选，仅在合成变化时触发）
- `write_pages`：写入所有新生成/更新的页面
- `post_validate`：后验校验（broken links, index coverage）

#### Lint 环节
- `lint_total`：完整 lint 流程
- `check_orphans`：孤儿页面检测（纯代码）
- `check_broken_links`：断链检测（纯代码）
- `check_missing_entities`：缺失实体检测（纯代码）
- `llm_contradictions`：跨页面矛盾检测（LLM）
- `llm_data_gaps`：数据缺口分析（LLM）

#### Query 环节
- `query_total`：查询端到端时间
- `read_index`：读取 index.md
- `find_relevant_pages`：相关性匹配（纯代码，含可选 graph 扩展）
- `read_pages`：读取候选页面内容
- `llm_answer`：LLM 合成答案
- `write_synthesis`：可选写入 synthesis 页面

---

## 3. 工具链改造方案

### 3.1 Token 计量改造

当前 `call_llm()` 仅返回字符串内容，**丢弃了 usage 元数据**。测速需改造为返回 `(content, usage_dict)` 二元组，或在函数内部将 usage 写入线程/进程安全的全局计量器。

**改造策略（非侵入式包装）**：

```python
# benchmark/metrics.py
import time
import threading
from dataclasses import dataclass, field
from typing import Dict, List

@dataclass
class LLMCallMetrics:
    call_id: str
    operation: str          # 如 "source_summary", "contradiction_check"
    start_time: float
    end_time: float = 0.0
    prompt_tokens: int = 0
    completion_tokens: int = 0
    total_tokens: int = 0
    model: str = ""

class MetricsCollector:
    def __init__(self):
        self.calls: List[LLMCallMetrics] = []
        self._lock = threading.Lock()
    
    def record(self, m: LLMCallMetrics):
        with self._lock:
            self.calls.append(m)
    
    def summary(self) -> Dict:
        # 按 operation 聚合，返回总耗时、总 token、平均 RTT 等
```

**Monkey-patch 方案（无需修改原文件）**：

在 benchmark 脚本中，运行时动态替换 `tools.ingest.call_llm`、`tools.lint.call_llm`、`tools.query.call_llm`，使其在调用原函数前后计时并提取 `response.usage`。

```python
import tools.ingest as ingest_mod

_original_call_llm = ingest_mod.call_llm

def _instrumented_call_llm(prompt: str, max_tokens: int = 8192) -> str:
    start = time.perf_counter()
    # 直接调用 litellm 以获取完整 response 对象
    from litellm import completion
    model = os.getenv("LLM_MODEL", "claude-3-5-sonnet-latest")
    response = completion(model=model, messages=[...], max_tokens=max_tokens)
    elapsed = time.perf_counter() - start
    
    collector.record(LLMCallMetrics(...))
    return response.choices[0].message.content

ingest_mod.call_llm = _instrumented_call_llm
```

### 3.2 子操作计时改造

对 `ingest.py`、`lint.py`、`query.py` 中的关键函数进行**上下文管理器包裹**，记录各阶段时间戳：

```python
# benchmark/timing.py
from contextlib import contextmanager
import time

@contextmanager
def timed_stage(name: str, report_dict: dict):
    t0 = time.perf_counter()
    yield
    t1 = time.perf_counter()
    report_dict[name] = t1 - t0
```

由于原文件较长（ingest.py 369 行，lint.py 453 行，query.py 213 行），**不建议直接大规模修改源码**，而是：

1. 在 benchmark 脚本中 `import` 并调用原工具函数，对外层进行计时；
2. 对 `ingest.py` 等源码做**最小侵入式修改**（插入 3–5 个 `timed_stage` 标记点），并通过 Git 暂存区管理，测速后可直接还原。

### 3.3 日志与输出标准化

所有测速结果统一输出为 JSON Lines（`.jsonl`）+ 汇总 Markdown 报告，便于后续可视化：

```
benchmark/
├── target.md          # 本规划文档
├── run_ingest.py      # ingest 测速入口
├── run_lint.py        # lint 测速入口
├── run_query.py       # query 测速入口
├── metrics.py         # 指标收集器
├── timing.py          # 计时工具
├── results/
│   ├── ingest_2026-04-21.jsonl
│   ├── lint_2026-04-21.jsonl
│   ├── query_2026-04-21.jsonl
│   └── summary.md
```

---

## 4. 测速代码架构设计

### 4.1 模块职责

| 模块 | 职责 |
|------|------|
| `benchmark/run_ingest.py` | 遍历 `examples/*.pdf` → 调用 `pdf2md` → 调用 `ingest` → 收集每篇论文的指标 |
| `benchmark/run_lint.py` | 在 wiki 构建完成后，重复运行 lint N 次（如 3 次）取平均，消除网络抖动 |
| `benchmark/run_query.py` | 设计 5–10 个典型查询问题，分别调用 `query.py`，记录响应时间与 token |
| `benchmark/metrics.py` | 全局 MetricsCollector，支持 LLMCallMetrics 的记录、聚合、导出 |
| `benchmark/timing.py` | `timed_stage` 上下文管理器，`Timer` 类支持嵌套计时 |
| `benchmark/report.py` | 读取 `.jsonl` 结果，生成 Markdown 汇总表与对比图表（可选 matplotlib） |

### 4.2 关键设计决策

#### 决策 1：Monkey-patch vs 源码修改
- **推荐**：Monkey-patch `call_llm` 以收集 token，对原文件零修改；仅在需要子阶段拆分时，对 `ingest.py` 插入最小标记（如 3 个 `timed_stage`）。
- **理由**：保持工具链可维护性，测速代码与业务逻辑解耦。

#### 决策 2：单篇串行 vs 批量并发
- **推荐**：ingest 保持**串行**（每篇 PDF 依次处理）。
- **理由**：ingest 会读写共享的 `index.md`、`overview.md`、`log.md`，并发会导致文件竞争；且 LLM API 通常有 RPM/TPM 限制，并发收益有限。

#### 决策 3：重复测量策略
- **推荐**：ingest 每篇只测 1 次（成本敏感）；lint 测 3 次取平均；query 每个问题测 3 次取平均。
- **理由**：ingest 涉及写入状态，重复测量需重置 wiki，成本高；lint 与 query 为只读或低副作用，适合多次采样。

#### 决策 4：PDF→MD 是否计入 ingest 耗时
- **推荐**：**分别计量**。`T_pdf2md` 作为独立指标；ingest 计时从 Markdown 文件就绪后开始。
- **理由**：pdf2md 是纯本地计算（或轻量 arxiv 下载），与 LLM 调用异构，分开便于定位瓶颈。

### 4.3 数据流

```
examples/*.pdf
    │
    ▼
[ pdf2md ] ──► raw/*.md ──► [ ingest ] ──► wiki/sources/*.md
                                              wiki/entities/*.md
                                              wiki/concepts/*.md
                                              wiki/index.md
                                              wiki/overview.md
                                              wiki/log.md
                                                  │
                    ┌─────────────────────────────┼─────────────────────────────┐
                    ▼                             ▼                             ▼
              [ run_ingest ]                [ run_lint ]                  [ run_query ]
              输出: ingest.jsonl            输出: lint.jsonl              输出: query.jsonl
                    │                             │                             │
                    └─────────────────────────────┴─────────────────────────────┘
                                                  │
                                                  ▼
                                          [ report.py ]
                                          输出: summary.md
```

---

## 5. 执行计划与里程碑

### Milestone 1：环境准备（约 10–15 分钟）
- [ ] 安装依赖：`pip install litellm networkx pymupdf4llm`
- [ ] 配置 LLM API Key 与模型环境变量（验证连通性）
- [ ] 验证 `python tools/health.py` 可正常运行（零 LLM 调用，作为基线）

### Milestone 2：测速框架搭建（约 20–30 分钟）
- [ ] 创建 `benchmark/metrics.py`、`benchmark/timing.py`
- [ ] 实现 Monkey-patch 包装器，验证 token 与 RTT 可正确采集
- [ ] 创建 `benchmark/run_ingest.py` 骨架，单篇 PDF 端到端跑通

### Milestone 3：全量 Ingest 与 Lint/Query 测速（约 30–60 分钟，视 LLM 速度而定）
- [ ] 将 `examples/` 24 篇 PDF 批量 pdf2md → ingest，记录 `ingest.jsonl`
- [ ] 执行 `run_lint.py` 3 轮，记录 `lint.jsonl`
- [ ] 设计并执行 5–10 个 query，记录 `query.jsonl`

### Milestone 4：报告生成（约 10 分钟）
- [ ] 运行 `benchmark/report.py` 生成 `benchmark/results/summary.md`
- [ ] 包含：各环节总耗时表、LLM 调用明细表、Token 消耗汇总、瓶颈分析

### Milestone 5：Wiki 交付（与测速并行）
- [ ] 确保 `wiki/` 目录完整构建，index.md / log.md / overview.md 符合规范
- [ ] 运行 `health` 与 `lint` 校验 wiki 质量

---

## 6. 风险与应对

| 风险 | 可能性 | 应对策略 |
|------|--------|---------|
| LLM API 调用失败 / 限流 | 中 | 在测速脚本中加入 `tenacity` 重试与指数退避；记录失败率 |
| `marker` / `pymupdf4llm` 解析复杂 PDF 失败 | 中 | 捕获异常，回退到 `pymupdf4llm`（轻量）；失败 PDF 跳过并记录 |
| ingest 生成的 wiki 页面过大导致后续 lint/query 变慢 | 低 | 监控每轮 lint/query 的输入 token 量，若异常增长则排查 |
| API 费用超出预期 | 中 | 先以 1–2 篇 PDF 做小规模试点，估算单篇成本后再全量跑 |

---

## 7. 附录：典型问题清单（Query 测速用）

以下问题用于 `run_query.py` 的标准化测试集：

1. `What are the main attack vectors in software supply chains discussed across all sources?`
2. `How does PyPI ecosystem security compare to NPM according to the wiki?`
3. `Summarize the detection techniques for malicious packages in open-source ecosystems.`
4. `What role do LLMs play in supply chain security based on the ingested papers?`
5. `List the key frameworks, tools, or datasets mentioned for analyzing software supply chains.`
6. `What contradictions exist between different papers regarding the effectiveness of static analysis?`
7. `Which papers discuss dependency confusion or dependency chaos, and what are their conclusions?`
8. `How is the concept of SBOM (Software Bill of Materials) used in risk assessment?`

---

> **下一步行动**：确认本规划后，进入 Milestone 1（环境准备）与 Milestone 2（测速框架编码）。
