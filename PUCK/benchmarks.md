# Benchmarks

Run with `python benchmarks/run_bench.py` (MVP set: 1, 2, 4, 6, 8 qubits) or `--extended` (adds 10–20).
Raw results are saved to `benchmarks/results/`. Fill in [`benchmarks/REPORT_TEMPLATE.md`](../benchmarks/REPORT_TEMPLATE.md)
for each device.

## Conditions (fixed workload)

* Circuit for *n* qubits: 20 layers of [H on every qubit → CX chain q0→q1→…→q(n−1) → RZ(0.3) on every qubit],
  then all qubits measured (implicit). Ops = 20·(3n−1); depth = 20·(…) as reported.
* Shots: 1,000 and 10,000; seed 1; one warm-up run, then 5 measured repeats (median reported).
* Each configuration runs in a fresh process; peak RSS is the whole Python process (≈20 MB is the interpreter +
  engine baseline); "Δ RSS" is the growth caused by the job.
* CPU utilisation = (user + system CPU time) / wall time (1.0 = one core). The engine is single-threaded;
  values slightly above 1.0 include interpreter/kernel accounting overhead.
* Wall time includes JSON encoding/decoding through ctypes; "engine" is the engine's own `runtime_ms`.

No fixed execution time is promised for arbitrary circuits: runtime grows roughly as ops × 2^n, plus a shot term.

## Development laptop — Intel Core i7-8565U @ 1.8 GHz, WSL2 Ubuntu 24.04, gcc 13.3 `-O3`

Measured 2026-10-09 (`benchmarks/results/bench-x86_64-20261009-112052.*`).

| qubits | ops | shots | wall median (ms) | engine (ms) | CPU util | peak RSS (MB) | Δ RSS (MB) | state |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 40 | 1,000 | 0.128 | 0.041 | 1.09 | 20.1 | 0.00 | 32 B |
| 1 | 40 | 10,000 | 0.489 | 0.378 | 1.09 | 20.4 | 0.32 | 32 B |
| 2 | 100 | 1,000 | 0.239 | 0.067 | 1.09 | 20.1 | 0.00 | 64 B |
| 2 | 100 | 10,000 | 0.678 | 0.520 | 1.09 | 20.4 | 0.32 | 64 B |
| 4 | 220 | 1,000 | 0.425 | 0.117 | 1.09 | 20.5 | 0.35 | 256 B |
| 4 | 220 | 10,000 | 1.053 | 0.726 | 1.09 | 20.6 | 0.39 | 256 B |
| 6 | 340 | 1,000 | 0.739 | 0.213 | 1.07 | 20.8 | 0.67 | 1 KiB |
| 6 | 340 | 10,000 | 1.427 | 0.905 | 1.09 | 20.9 | 0.71 | 1 KiB |
| **8** | **460** | **1,000** | **1.365** | **0.549** | 1.09 | 21.0 | 0.85 | 4 KiB |
| **8** | **460** | **10,000** | **2.175** | **1.393** | 1.09 | 21.0 | 0.84 | 4 KiB |
| 10 | 580 | 1,000 | 3.398 | 1.995 | 1.09 | 21.3 | 1.30 | 16 KiB |
| 12 | 700 | 1,000 | 12.08 | 8.97 | 1.09 | 22.2 | 1.99 | 64 KiB |
| 14 | 820 | 1,000 | 35.79 | 32.44 | 1.09 | 22.7 | 2.22 | 256 KiB |
| 16 | 940 | 1,000 | 124.1 | 120.7 | 1.09 | 25.5 | 5.16 | 1 MiB |
| 18 | 1,060 | 1,000 | 581.3 | 578.1 | 1.09 | 37.0 | 16.6 | 4 MiB |
| 20 | 1,180 | 1,000 | 3,322 | 3,318 | 1.09 | 79.2 | 58.8 | 16 MiB |

Kernel optimisation history on the same machine (20 qubits, 1,180 ops): first implementation 10.2 s / 391 MB
peak → bit-insertion indexing, permutation fast path and output caps 3.3 s / 79 MB.

## Raspberry Pi 4B 4 GB

**Not yet measured** — requires the physical device. Run `python3 benchmarks/run_bench.py --extended` on the Pi
and add the table here using the template.
