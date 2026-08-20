# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 11 | 0.20 | 30000 | 54000 | 54000 | 6.2 | 0.0% |
| 50 | 13 | 0.23 | 36000 | 56000 | 56000 | 7.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.16x** (23% of linear) |
| P95 latency | **1.04x** |
| Effective concurrency at 50 users | 7.8 vs `--parallel 4` slots (occupancy/slot ratio 1.96) |

**Saturated.** Throughput delivered only 1.16x for 5x the offered load, and effective concurrency (7.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.04x vs 1.16x), so this server still has headroom at 50 users.

> **Small sample.** Only 11 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is saturated somewhere at or below 50 simulated users. Increasing the
offered load from 10 to 50 users (5x) raised delivered throughput only from 0.20
to 0.23 RPS (1.16x), while effective concurrency reached 7.8 for only four decode
slots. The most convincing evidence is the server-side measurement: peak
`n_busy_slots_per_decode` reached 3.88 of 4 slots (97%) and 46 requests were
deferred. This proves that the slots were full and excess work was queued. The P95
increase from 54 to 56 seconds is small, but the run completed only 13 requests
and excludes requests still queued when Locust stopped, so throughput scaling and
server gauges are more reliable than these sample percentiles.

The first server knob I would test to raise goodput at the latency SLO is increasing
`--parallel` modestly from 4 to 8, provided memory usage remains safe. All four
current slots are already busy and requests are deferred, so adding slots directly
targets the observed bottleneck. I would validate this with the same load and
metrics tests, because extra slots can also increase per-request decode contention;
I would keep the change only if more requests meet the latency SLO.
