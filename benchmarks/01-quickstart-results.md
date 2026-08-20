# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 46690 | 901 / 6975 | 132.7 / 155.4 | 8820 / 15406 / 15406 | 7.5 |
| UD-Q2_K_XL | 2.24 | 20275 | 1240 / 12049 | 171.2 / 203.3 | 11433 / 24859 / 24859 | 5.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.29x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

On this machine, `UD-Q2_K_XL` is 0.73 GB (about 24.6%) smaller than
`UD-Q4_K_XL`, but the smaller model does not improve inference speed. Its median
decode rate is only 5.8 tok/s compared with 7.5 tok/s for Q4, so Q2 is about
22.7% slower (equivalently, Q4 is about 1.29x faster). Q2 also has a worse median
TTFT (1240 ms versus 901 ms) and a worse median end-to-end latency (11433 ms
versus 8820 ms). This suggests that inference on this machine is compute-limited:
the additional dequantization cost of the heavily quantized Q2 format outweighs
the benefit of reading fewer model bytes. Based on the measured size and speed,
Q4 is the better default choice on this machine; Q2 is worthwhile only when the
0.73 GB memory/disk saving is necessary. Answer-quality comparison has not yet
been performed, so no quality advantage is claimed here.
