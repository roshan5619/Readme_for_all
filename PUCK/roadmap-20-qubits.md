# Roadmap: from 8 to 20 qubits

The MVP is configured for **8 qubits**. Nothing in the design is specific to 8: the engine is compiled with a
ceiling of 24 qubits and the limit is a runtime setting (`max_qubits` in `puck.toml` / `PUCK_MAX_QUBITS`).
Raising it is a configuration change **after** the checks below pass on the target device.

## What changes with qubit count

| Qubits | State vector (complex128) | Working memory per job* |
|---:|---:|---:|
| 8 | 4 KiB | < 0.1 MiB |
| 12 | 64 KiB | ~0.2 MiB |
| 16 | 1 MiB | ~3 MiB |
| 20 | 16 MiB | ~48 MiB |
| 24 | 256 MiB | ~0.8 GiB |

\* state + sampling table + aggregation buffer (+ a second state copy for mid-circuit measurement).

Measured on the development laptop (Intel i7-8565U, WSL2, single thread, fixed 20-layer H/CX/RZ workload — see
[benchmarks.md](benchmarks.md)): 8 qubits ≈ 1.4 ms, 16 qubits ≈ 0.12 s, 20 qubits ≈ 3.3 s, peak RSS ≈ 80 MB.
A Raspberry Pi 4 is expected to be several times slower; **measure it** before choosing limits.

## Steps

1. **Benchmark on the Pi**: `python benchmarks/run_bench.py --extended` and record results.
2. **Set limits from data**: pick `max_qubits`, `max_gates`, `max_depth` and `job_timeout_ms` so that the worst
   accepted job finishes within the timeout with margin. Example (to be confirmed): 20 qubits, 500 ops, 30 s.
3. **Multi-core kernels**: add OpenMP `parallel for` over sub-blocks in `qp_apply_matrix` (enabled only above
   ~14 qubits; the Pi 4 has 4 cores). Expected 2.5–3.5× on the Pi; keep `workers=1` so jobs do not compete.
4. **Gate fusion**: merge consecutive 1-qubit gates on the same wire into one 2×2 matrix before execution (the
   benchmark circuit would drop from 3 to 2 passes per layer).
5. **Output size**: keep `max_outcomes` (4096) and `max_statevector_qubits` (14) — returning 2^20 entries to a
   browser is not useful; the Studio already shows a truncation notice.
6. **Memory guard**: the systemd unit caps the service at 1 GiB; 20 qubits fits comfortably. Do not go beyond
   22 qubits on a 4 GB Pi without lowering concurrency.
7. **UI**: the qubit selector follows the device limit automatically; the canvas scrolls vertically.
8. **Tests**: correctness tests already run circuits up to 8 qubits against independent references and the
   API tests exercise 20-qubit jobs (timeout/cancel). Add reference checks at 12–16 qubits when raising the limit.

## Known non-goals for the roadmap

* No noise models, no pulse-level simulation, no hardware execution (this is a simulator).
* No GPU path — the Pi 4 GPU is not useful for complex128 linear algebra.
