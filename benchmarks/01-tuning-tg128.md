# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.3 | 93% |
| 2 | 9.0 | 100% |
| 4 | 8.6 | 96% |
| 8 | 7.9 | 88% |
| 16 | 7.6 | 84% |

**Best**: `-t 2` at 9.0 tok/s
**Slowest tested**: `-t 16` at 7.6 tok/s (1.18x spread)
**Against the physical-core default** (`-t 4`, 8.6 tok/s): 1.04x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation

The knee of this curve is at 2 threads, where decode reaches the best measured
rate of 9.0 tok/s. Increasing the thread count to the 4 physical cores does not
improve throughput; instead, performance falls slightly to 8.6 tok/s (about 4%
below the best). Oversubscribing the CPU makes the decline clearer: 8 threads
produce 7.9 tok/s and 16 threads produce only 7.6 tok/s, about 16% below the
best result. Therefore, more threads are not useful for this workload on this
machine. The likely explanation is that decode is constrained by memory access
and synchronization rather than a lack of CPU workers. Additional threads compete
for shared memory bandwidth and add scheduling/synchronization overhead. Since
the model also runs with GPU offload (`ngl=99`), CPU thread scaling is limited
further. I would use `-t 2` for subsequent runs; the improvement over the default
`-t 4` is modest but repeatable in this sweep (1.04x).
