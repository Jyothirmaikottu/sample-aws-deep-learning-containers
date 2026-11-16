# Reference Results

Numbers from one run of this sample on 2026-09-30, `inference-charts` 0.2.7 at commit
`21fda29`, inference-perf v0.6.1. Use them as a reference point, not as a guarantee; your
numbers will vary with image versions, Region, and node placement.

Workload: `Qwen/Qwen3-8B` (vLLM) and `unsloth/Qwen3-8B-GGUF` (llama.cpp), fixed 512 input /
128 output tokens, streaming, `ignore_eos: true`. Graviton is `m8g.4xlarge`, GPU is
`g6e.xlarge` (1x L40S). The client ran on a separate `m7i.xlarge`.

Metrics:

- **Decode, 1 user (t/s):** from `bench/curl-decode.sh`. It times a 1-token and a 129-token
  request with no other load and divides 128 by the difference. For llama.cpp this matches
  the server's `predicted_per_second` within 0.5%.
- **Output TPS:** total output tokens over run time, prefill included.
- **TTFT and ITL:** p50 from inference-perf. Per-user t/s is 1/ITL.

## Summary

| Engine | Device | Precision | Decode, 1 user (t/s) | TTFT, 1 user (s) | Peak output TPS (concurrency) | Peak memory |
|---|---|---|---|---|---|---|
| llama.cpp | Graviton4 | Q4_K_M | 30.8 | 6.42 | 14.2 (4) | 13.3 GiB RSS |
| llama.cpp | Graviton4 | Q8_0 | 28.7 | 4.82 | 18.7 (4) | 19.7 GiB RSS |
| llama.cpp | Graviton4 | BF16 | 2.37 | 188 | not swept | 16.1 GiB RSS |
| vLLM 0.30.0 | Graviton4 | BF16 | 4.33 | 4.80 | 17.3 (8) | 40.7 GiB RSS |
| llama.cpp | L40S | Q4_K_M | 125.8 | 0.11 | 269.4 (4) | 5.5 GiB VRAM |
| llama.cpp | L40S | Q8_0 | 79.2 | 0.11 | 200.6 (4) | 8.6 GiB VRAM |
| llama.cpp | L40S | BF16 | 46.3 | 0.11 | 136.5 (4) | 15.3 GiB VRAM |
| vLLM | L40S | BF16 | 46.2 | 0.05 | 1,279.6 (64) | ~43 GiB VRAM (reserved) |

Notes:

- `llama-server` ran with its default 4 slots (see the README). Past concurrency 4, the
  llama.cpp numbers are requests waiting in a queue, not more batching, so llama.cpp peak
  TPS is likely understated, mostly on the GPU.
- llama.cpp BF16 on Graviton was not swept. One 512/128 request takes about 4 minutes, and
  188 s of that is prefill.
- About half of vLLM's 40.7 GiB on Graviton is the 20 GiB KV cache set in the values file.
  The GPU vLLM number is what `--gpu-memory-utilization 0.9` reserves, not what it used.
- The Graviton BF16 memory number is from a single request. Q8_0 had 4 requests in flight,
  which is why it shows more memory than BF16.

## Cost per 1M output tokens

Based on us-west-2 on-demand prices at the time of the run: `m8g.4xlarge` $0.718/hr,
`g6e.xlarge` $1.861/hr. Check current pricing for your Region. Cost is the hourly price
divided by output tokens per hour at peak, so it assumes the node is fully busy. At 10%
utilization, multiply by 10.

| Setup | Output TPS | p50 TTFT | $ per 1M output tokens |
|---|---|---|---|
| vLLM L40S BF16, conc 64 | 1,279.6 | 0.48 s | 0.40 |
| llama.cpp L40S Q4_K_M, conc 4 | 269.4 | 0.70 s | 1.92 |
| llama.cpp L40S Q8_0, conc 4 | 200.6 | 0.51 s | 2.58 |
| llama.cpp L40S BF16, conc 4 | 136.5 | 0.71 s | 3.79 |
| llama.cpp Graviton Q8_0, conc 4 | 18.7 | 19.4 s | 10.67 |
| vLLM Graviton BF16, conc 8 | 17.3 | 17.4 s | 11.53 |
| llama.cpp Graviton Q4_K_M, conc 4 | 14.2 | 25.8 s | 14.05 |

vLLM on the L40S reaches 1,762 TPS (about $0.29) at 32 req/s, but p50 TTFT at that rate is
19 s, so the table uses concurrency 64. The Graviton points all sit at 17 to 26 s TTFT, so
they only make sense for jobs that can wait that long.

## Observations

- For a single user, the quant order is the same on both devices: Q4_K_M, then Q8_0, then
  BF16. On the GPU the gaps are large (126 / 79 / 46 t/s) because decode is bound by memory
  bandwidth. On Graviton, Q4_K_M and Q8_0 are close, and BF16 drops to 2.4 t/s because
  llama.cpp has no fast BF16 path on Arm.
- Under load on Graviton, Q8_0 beats Q4_K_M (18.7 vs 14.2 TPS). Its prefill is faster (4.8
  vs 6.4 s TTFT), and prefill dominates once requests queue up.
- Prefill is the main limit on Graviton, not decode. The decode gap with the GPU is about
  4x, but the TTFT gap is 40x to 100x.
- vLLM is the engine here that scales with load. On the L40S it goes from 46 t/s for one
  user to 1,280 TPS at concurrency 64.
- On the L40S, llama.cpp BF16 and vLLM BF16 decode at the same speed (46.3 vs 46.2 t/s), a
  useful check that the setup is sound. The 9x difference under load comes from batching.

## Per-cell sweeps

llama.cpp Graviton Q4_K_M

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 6.42 | 33.1 | 30.2 | 11.6 | 10.95 |
| conc 2 | 10/0 | 12.87 | 62.1 | 16.1 | 11.9 | 21.34 |
| conc 4 | 16/0 | 25.78 | 73.1 | 13.7 | 14.2 | 36.00 |
| conc 8 | 16/0 | 57.50 | 73.3 | 13.6 | 13.5 | 75.82 |

llama.cpp Graviton Q8_0

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 4.82 | 36.3 | 27.5 | 12.9 | 9.84 |
| conc 2 | 10/0 | 9.69 | 63.9 | 15.6 | 13.9 | 18.48 |
| conc 4 | 16/0 | 19.38 | 55.6 | 18.0 | 18.7 | 27.32 |
| conc 8 | 16/0 | 46.48 | 55.7 | 18.0 | 18.7 | 54.14 |

vLLM Graviton BF16

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 4.80 | 241.4 | 4.1 | 3.4 | 37.04 |
| conc 2 | 10/0 | 8.13 | 255.6 | 3.9 | 6.0 | 42.50 |
| conc 4 | 16/0 | 12.58 | 258.5 | 3.9 | 11.1 | 45.42 |
| conc 8 | 16/0 | 17.41 | 269.5 | 3.7 | 17.3 | 57.58 |

llama.cpp L40S Q4_K_M

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.11 | 8.1 | 123.3 | 110.8 | 1.14 |
| conc 4 | 16/0 | 0.70 | 10.0 | 99.6 | 269.4 | 1.99 |
| conc 16 | 64/0 | 6.72 | 10.2 | 97.7 | 254.7 | 8.03 |
| conc 32 | 128/0 | 15.23 | 10.5 | 95.0 | 247.3 | 16.58 |
| conc 64 | 256/0 | 32.91 | 11.1 | 89.9 | 237.9 | 34.66 |

llama.cpp L40S Q8_0

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.11 | 12.8 | 78.1 | 73.1 | 1.74 |
| conc 4 | 16/0 | 0.51 | 14.9 | 66.9 | 200.6 | 2.64 |
| conc 16 | 64/0 | 8.68 | 15.4 | 64.7 | 192.0 | 10.66 |
| conc 32 | 128/0 | 19.40 | 15.4 | 65.0 | 188.7 | 21.60 |
| conc 64 | 256/0 | 42.10 | 16.0 | 62.6 | 184.1 | 44.49 |

llama.cpp L40S BF16

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.11 | 21.7 | 46.0 | 44.3 | 2.88 |
| conc 4 | 16/0 | 0.71 | 24.5 | 40.8 | 136.5 | 3.83 |
| conc 16 | 64/0 | 12.30 | 24.7 | 40.5 | 132.3 | 15.45 |
| conc 32 | 128/0 | 27.96 | 25.0 | 40.0 | 130.4 | 31.41 |
| conc 64 | 256/0 | 60.50 | 25.6 | 39.1 | 127.2 | 64.25 |

vLLM L40S BF16, concurrency sweep

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.05 | 21.8 | 45.9 | 42.6 | 2.82 |
| conc 4 | 16/0 | 0.12 | 23.5 | 42.6 | 160.6 | 3.14 |
| conc 16 | 64/0 | 0.29 | 25.0 | 40.0 | 541.7 | 3.72 |
| conc 32 | 128/0 | 0.44 | 27.6 | 36.2 | 874.3 | 4.61 |
| conc 64 | 256/0 | 0.48 | 32.6 | 30.7 | 1279.6 | 6.34 |

vLLM L40S BF16, Poisson rate sweep

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| rate 2 (achieved 2.1) | 120/0 | 0.09 | 24.0 | 41.7 | 255.0 | 3.29 |
| rate 8 (achieved 7.9) | 480/0 | 0.14 | 30.3 | 33.0 | 992.7 | 4.99 |
| rate 16 (achieved 15.3) | 959/1 | 2.84 | 130.1 | 7.7 | 1649.3 | 20.84 |
| rate 32 (achieved 16.2) | 1920/0 | 19.20 | 160.5 | 6.2 | 1762.0 | 37.57 |

## Not covered

- vLLM with GGUF weights. vLLM moved GGUF support to `vllm-gguf-plugin`, which the DLC
  images used here do not include.
- vLLM W8A8 on Graviton, the closer 8-bit comparison against llama.cpp Q8_0.
- llama.cpp with more than 4 parallel slots.
