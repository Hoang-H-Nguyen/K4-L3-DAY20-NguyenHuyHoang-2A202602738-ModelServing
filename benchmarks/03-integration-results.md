# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 9259.9 | 9259.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 464.1 | 464.2 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 1139.3 | 1139.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **3621.1** · total **3621.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput is more useful than raw throughput because it specifically filters for requests that meet the **TTFT** (Throughput-to-Temperature Failure) and **TPOT** (Throughput-to-Power Overhead) targets.

While raw throughput measures the total number of requests per second, it ignores the constraints of the system. Goodput ensures that the system remains stable and efficient by only counting requests

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory by storing the key-value cache (KV cache) in non-contiguous pages, which allows the GPU to utilize more memory than would be available if the cache were stored contiguously.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound** and **decode is memory-bandwidth-bound**.

In this context, the split allows the system to leverage the specific characteristics of each operation:
*   **Prefill:** Since it is compute-bound, splitting it onto separate pools (e.g., via RadixAttention keys cached KV by token prefix) allows the engine to skip the heavy compute work


## Which N16-N19 pieces are real

N16 Cloud/IaC: stub. N17 data pipeline: stub. N18 lakehouse: stub.
N19 vector + features: stub. N20 serving is real (`llama-server`); this run uses
the documented toy corpus and keyword-overlap retrieval fallback. The dominant
stage is LLM at 3621.1 ms (100% of the 3621.2 ms total), which matches the
expectation that generation dominates on this CPU. To halve latency, I would
attack the LLM stage first by reducing output-token budget and tuning CPU/thread
configuration; embed and retrieval are both effectively 0 ms here.
