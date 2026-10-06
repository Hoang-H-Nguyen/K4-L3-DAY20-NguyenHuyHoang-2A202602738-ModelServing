# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 19.1 | 64% |
| 6 | 30.1 | 100% |
| 12 | 28.6 | 95% |
| 16 | 22.9 | 76% |
| 32 | 17.8 | 59% |

**Best**: `-t 6` at 30.1 tok/s
**Slowest tested**: `-t 32` at 17.8 tok/s (1.69x spread)
**Against the physical-core default** (`-t 12`, 28.6 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

The knee is around 6 threads: throughput peaks at 30.1 tok/s, already slightly
above the 12-thread physical-core default at 28.6 tok/s. Adding threads beyond
the knee hurts decode throughput: 16 threads fall to 22.9 tok/s and the
oversubscribed 32-thread run reaches only 17.8 tok/s. Decode is sharing the
same memory bandwidth, while extra threads also add scheduling and cache
contention; they do not create more useful work. This CPU-only run therefore
benefits from a moderate thread count rather than using every logical core.
