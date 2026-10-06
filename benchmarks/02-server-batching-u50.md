# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 25 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 18687 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak `n_busy_slots_per_decode` was 3.97 of 4 slots (99%), with
`requests_processing=4` and `requests_deferred=45`. This is direct evidence that
continuous batching was active and the scheduler kept all slots busy. It is much
smaller than the report's effective concurrency of 41.0 because that number is
Little's-Law occupancy and includes queued requests; the server gauge is the
correct measure of simultaneous decode-slot utilisation.
