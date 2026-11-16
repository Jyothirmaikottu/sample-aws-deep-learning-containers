# Reference Results

Numbers from one run of this sample on 2026-11-16, `inference-charts` 0.2.7 at commit
`21fda29`, inference-perf v0.6.1. Use them as a reference point, not as a guarantee; your
numbers will vary with image versions, Region, and node placement.

Images: `llama-cpp-arm64:server-cpu-v1.0.6` (llama.cpp build b10964),
`llama-cpp:server-cuda-v1.0.6`, `vllm-arm64:server-cpu-v1.1.0` (vLLM 0.33.0), and
`vllm:server-cuda-v2.8.1` (vLLM 0.33.0). llama.cpp ran with `--parallel 8 --ctx-size 8192`.

Workload: `Qwen/Qwen3-8B` (vLLM) and `unsloth/Qwen3-8B-GGUF` (llama.cpp), fixed 512 input /
128 output tokens, streaming, `ignore_eos: true`. Graviton is `m8g.4xlarge`, GPU is
`g6e.xlarge` (1x L40S). The client ran on a separate `m7i.xlarge`, and all nodes were in one
Availability Zone.

Metrics:

- **Decode, 1 user (t/s):** from `bench/curl-decode.sh`, mean of 3 runs. It times a 1-token
  and a 129-token request with no other load and divides 128 by the difference. For
  llama.cpp this matches the server's `predicted_per_second` within 0.5%.
- **Output TPS:** total output tokens over run time, prefill included.
- **TTFT and ITL:** p50 from inference-perf. Per-user t/s is 1/ITL.

## Summary

| Engine | Device | Precision | Decode, 1 user (t/s) | TTFT, 1 user (s) | Peak output TPS (concurrency) | Peak memory |
|---|---|---|---|---|---|---|
| llama.cpp | Graviton4 | Q4_K_M | 31.3 | 6.41 | 13.9 (8) | 14.2 GiB RSS |
| llama.cpp | Graviton4 | Q8_0 | 26.9 | 4.78 | 20.1 (8) | 20.7 GiB RSS |
| llama.cpp | Graviton4 | BF16 | 2.37 | 188 | not swept | 16.5 GiB RSS |
| vLLM 0.33.0 | Graviton4 | BF16 | 4.57 | 4.57 | 18.1 (8) | 40.6 GiB RSS |
| llama.cpp | L40S | Q4_K_M | 126.2 | 0.11 | 406.7 (16) | 6.1 GiB VRAM |
| llama.cpp | L40S | Q8_0 | 79.2 | 0.10 | 331.0 (16) | 9.2 GiB VRAM |
| llama.cpp | L40S | BF16 | 46.4 | 0.11 | 233.5 (16) | 15.8 GiB VRAM |
| vLLM 0.33.0 | L40S | BF16 | 46.2 | 0.05 | 1,280.6 (64) | ~42 GiB VRAM (reserved) |

Notes:

- `llama-server` ran 8 slots (see the README). Past concurrency 8, the llama.cpp numbers are
  requests waiting in a queue, not more batching.
- llama.cpp BF16 on Graviton was not swept. One 512/128 request takes about 4 minutes, and
  188 s of that is prefill.
- About half of vLLM's 40.6 GiB on Graviton is the 20 GiB KV cache set in the values file.
  The GPU vLLM number is what `--gpu-memory-utilization 0.9` reserves, not what it used.
- The Graviton BF16 memory number is from a single request. Q8_0 had 8 requests in flight,
  which is why it shows more memory than BF16.

## Cost per 1M output tokens

Based on us-west-2 on-demand prices: `m8g.4xlarge` $0.718/hr, `g6e.xlarge` $1.861/hr. Check
current pricing for your Region. Cost is the hourly price divided by output tokens per hour
at peak, so it assumes the node is fully busy. At 10% utilization, multiply by 10.

| Setup | Output TPS | p50 TTFT | $ per 1M output tokens |
|---|---|---|---|
| vLLM L40S BF16, conc 64 | 1,280.6 | 0.50 s | 0.40 |
| llama.cpp L40S Q4_K_M, conc 16 | 406.7 | 2.85 s | 1.27 |
| llama.cpp L40S Q8_0, conc 16 | 331.0 | 3.42 s | 1.56 |
| llama.cpp L40S BF16, conc 16 | 233.5 | 4.75 s | 2.21 |
| llama.cpp Graviton Q8_0, conc 8 | 20.1 | 23.5 s | 9.92 |
| vLLM Graviton BF16, conc 8 | 18.1 | 17.0 s | 11.02 |
| llama.cpp Graviton Q4_K_M, conc 8 | 13.9 | 37.6 s | 14.35 |

vLLM on the L40S reaches 1,723 TPS (about $0.30) at a requested 32 req/s, but the achieved
rate tops out at 16 req/s and p50 TTFT is 19.7 s, so the table uses concurrency 64. The
Graviton points all sit at 17 to 38 s TTFT, so they only make sense for jobs that can wait
that long.

## Observations

- For a single user, the quant order is the same on both devices: Q4_K_M, then Q8_0, then
  BF16. On the GPU the gaps are large (126 / 79 / 46 t/s) because decode is bound by memory
  bandwidth. On Graviton, Q4_K_M and Q8_0 are close (31.3 vs 26.9 t/s), and BF16 drops to
  2.4 t/s because llama.cpp has no fast BF16 path on Arm.
- Under load on Graviton, Q8_0 beats Q4_K_M (20.1 vs 13.9 TPS). Its prefill is faster (4.8
  vs 6.4 s TTFT), and prefill dominates once requests queue up.
- Going from 4 to 8 llama.cpp slots raised GPU peak throughput by about 50 to 70% (Q4_K_M
  407 vs 269 TPS in the 4-slot run) but barely moved Graviton (Q8_0 20.1 vs 18.7, Q4_K_M
  13.9 vs 14.2). On Graviton the CPU is already saturated at 4 requests.
- Prefill is the main limit on Graviton, not decode. For llama.cpp the decode gap with the
  GPU is 3x to 4x, but the single-user TTFT gap is about 50x to 60x (about 90x for vLLM).
- vLLM is the engine here that scales with load. On the L40S it goes from 46 t/s for one
  user to 1,281 TPS at concurrency 64, about 3x the best llama.cpp GPU result.
- On the L40S, llama.cpp BF16 and vLLM BF16 decode at the same speed (46.4 vs 46.2 t/s), a
  useful check that the setup is sound. The 5.5x difference under load comes from batching.

## Per-cell sweeps

llama.cpp Graviton Q4_K_M

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 6.41 | 31.4 | 31.9 | 11.9 | 10.72 |
| conc 2 | 10/0 | 12.27 | 59.5 | 16.8 | 12.5 | 20.41 |
| conc 4 | 16/0 | 24.54 | 118.6 | 8.4 | 13.6 | 37.57 |
| conc 8 | 16/0 | 37.62 | 144.9 | 6.9 | 13.9 | 67.94 |

llama.cpp Graviton Q8_0

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 4.78 | 36.5 | 27.4 | 12.7 | 10.00 |
| conc 2 | 10/0 | 9.01 | 61.0 | 16.4 | 14.6 | 17.53 |
| conc 4 | 16/0 | 18.07 | 119.0 | 8.4 | 16.9 | 30.17 |
| conc 8 | 16/0 | 23.53 | 94.4 | 10.6 | 20.1 | 50.74 |

vLLM Graviton BF16

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 4.57 | 227.4 | 4.4 | 3.7 | 34.92 |
| conc 2 | 10/0 | 6.99 | 241.7 | 4.1 | 6.1 | 40.35 |
| conc 4 | 16/0 | 12.27 | 244.3 | 4.1 | 11.7 | 43.38 |
| conc 8 | 16/0 | 17.00 | 254.4 | 3.9 | 18.1 | 55.13 |

llama.cpp L40S Q4_K_M

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.11 | 8.1 | 123.2 | 110.9 | 1.14 |
| conc 4 | 16/0 | 0.21 | 10.5 | 94.8 | 291.0 | 1.76 |
| conc 16 | 64/0 | 2.85 | 13.2 | 75.5 | 406.7 | 5.05 |
| conc 32 | 128/0 | 8.24 | 13.7 | 72.9 | 388.5 | 10.55 |
| conc 64 | 256/0 | 19.87 | 14.9 | 67.2 | 370.4 | 22.33 |

llama.cpp L40S Q8_0

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.10 | 12.8 | 78.3 | 64.7 | 1.73 |
| conc 4 | 16/0 | 0.31 | 14.9 | 67.1 | 219.0 | 2.32 |
| conc 16 | 64/0 | 3.42 | 17.6 | 56.8 | 331.0 | 6.16 |
| conc 32 | 128/0 | 10.00 | 18.2 | 54.8 | 318.4 | 12.91 |
| conc 64 | 256/0 | 23.86 | 19.3 | 51.9 | 307.8 | 26.93 |

llama.cpp L40S BF16

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.11 | 21.8 | 45.9 | 44.1 | 2.89 |
| conc 4 | 16/0 | 0.16 | 25.1 | 39.9 | 137.2 | 3.65 |
| conc 16 | 64/0 | 4.75 | 27.2 | 36.8 | 233.5 | 8.76 |
| conc 32 | 128/0 | 13.95 | 27.8 | 35.9 | 225.8 | 18.10 |
| conc 64 | 256/0 | 33.07 | 28.9 | 34.6 | 220.2 | 37.37 |

vLLM L40S BF16, concurrency sweep

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| conc 1 | 8/0 | 0.05 | 21.8 | 45.9 | 44.8 | 2.82 |
| conc 4 | 16/0 | 0.10 | 23.5 | 42.5 | 158.5 | 3.15 |
| conc 16 | 64/0 | 0.35 | 25.0 | 39.9 | 536.1 | 3.75 |
| conc 32 | 128/0 | 0.46 | 27.6 | 36.2 | 896.8 | 4.63 |
| conc 64 | 256/0 | 0.50 | 32.7 | 30.6 | 1280.6 | 6.43 |

vLLM L40S BF16, Poisson rate sweep

| Load | OK/fail | TTFT (s) | ITL (ms) | Per-user t/s | Output TPS | Req latency (s) |
|---|---|---|---|---|---|---|
| rate 2 (achieved 1.7) | 120/0 | 0.09 | 23.8 | 42.0 | 218.5 | 3.24 |
| rate 8 (achieved 8.5) | 480/0 | 0.14 | 31.5 | 31.7 | 1039.2 | 5.49 |
| rate 16 (achieved 16.0) | 960/0 | 5.12 | 129.5 | 7.7 | 1669.6 | 21.78 |
| rate 32 (achieved 16.0) | 1920/0 | 19.67 | 163.1 | 6.1 | 1723.3 | 38.27 |

## Not covered

- vLLM with GGUF weights. vLLM moved GGUF support to `vllm-gguf-plugin`, which the DLC
  images used here do not include.
- vLLM W8A8 on Graviton, the closer 8-bit comparison against llama.cpp Q8_0.
