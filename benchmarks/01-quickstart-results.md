# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 4110 | 69 / 80 | 7.0 / 7.4 | 507 / 549 / 549 | 143.3 |
| UD-Q2_K_XL | 0.39 | 3064 | 72 / 79 | 7.7 / 8.3 | 559 / 600 / 600 | 129.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.10x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Trên máy này, UD-Q2_K_XL nhỏ hơn Q4_K_M khoảng 0.11 GB, tương đương 22%, nhưng decode chậm hơn khoảng 9.4% (129.8 so với 143.3 tok/s). TTFT cũng tương đương, còn TPOT của 2-bit cao hơn. Khi hỏi cùng một câu, chất lượng hai bản gần như giống nhau. Vì vậy Q2 chỉ đáng dùng nếu ưu tiên tiết kiệm dung lượng; Q4 đáng dùng hơn cho tốc độ.
