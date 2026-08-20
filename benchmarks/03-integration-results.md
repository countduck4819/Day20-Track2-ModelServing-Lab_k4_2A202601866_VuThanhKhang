# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.3 | 8588.3 | 8588.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 6573.1 | 6573.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 7021.9 | 7022.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **7394.4** · total **7394.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- N16 Cloud/IaC: stubbed; the service runs on localhost rather than a deployed
  cluster or Compose stack.
- N17 Data pipeline: stubbed; documents come from an in-memory list rather than a
  scheduled ingestion pipeline.
- N18 Lakehouse: stubbed; `TOY_DOCS` stands in for a Delta/Iceberg table.
- N19 Vector + features: stubbed; retrieval uses keyword overlap rather than a real
  vector index or feature store.
- N20 Serving: real; the pipeline calls the local llama-server through its
  OpenAI-compatible HTTP endpoint.

The LLM was the expected dominant stage, but its 7394.4 ms mean accounted for
essentially 100% of the 7394.7 ms total, an even larger share than expected. To
halve end-to-end latency, I would optimize the LLM stage first because embedding
and retrieval together take only about 0.2 ms. The first candidates would be a
faster serving configuration/model and a smaller generation budget; optimizing
the sub-millisecond retrieval path cannot produce a meaningful 2x improvement.
