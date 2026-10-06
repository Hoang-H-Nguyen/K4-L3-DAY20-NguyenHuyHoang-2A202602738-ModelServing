# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 119 | 2.02 | 2200 | 21000 | 26000 | 8.2 | 0.0% |
| 50 | 203 | 3.45 | 13000 | 15000 | 15000 | 41.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.71x** (34% of linear) |
| P95 latency | **0.71x** |
| Effective concurrency at 50 users | 41.0 vs `--parallel 4` slots (occupancy/slot ratio 10.26) |

**Saturated.** Throughput delivered only 1.71x for 5x the offered load, and effective concurrency (41.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (0.71x vs 1.71x), so this server still has headroom at 50 users.

## Your reading

The server reaches saturation by 50 users. Throughput rises only 1.71x when
offered load rises 5x, while the live metrics show all four slots occupied
(`n_busy_slots_per_decode=3.97/4`, `requests_processing=4`) and 45 deferred
requests. Effective concurrency reaches 41.0 because it includes queued work;
the deferred counter is direct evidence that the extra latency is queue time.
To raise goodput at an SLO, I would tune CPU/threading first, since decode is
already using every slot; increasing `--parallel` alone would lengthen the queue.
