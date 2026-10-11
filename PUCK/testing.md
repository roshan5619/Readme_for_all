# Testing

## Commands

```bash
# C engine unit tests (169 checks) + Bell self-test
make -C engine test

# Python: quantum correctness, API, SDK, PennyLane, QASM, config, launcher
~/.venvs/quanverge-puck/bin/python -m pytest tests -q

# Frontend unit tests + typecheck
cd apps/studio && npm test && npm run typecheck

# Browser end-to-end (needs a running Puck and `pip install playwright`; uses installed Edge/Chrome)
./scripts/run-laptop.sh --no-browser &        # in WSL / Linux / macOS
python tests/e2e/studio_e2e.py --channel msedge --shots ./e2e-shots
```

## What is covered

| Suite | File(s) | Covers |
|---|---|---|
| C unit | `engine/tests/test_engine.c` | Every gate unitary; every gate's kernel vs dense matrix–vector product on 5 qubits; Bell/GHZ-8; little-endian order and CX direction; seed reproducibility; mid-circuit measure + reset; 15 validation error codes; cancellation; output caps; QASM import, errors with line numbers, gate-definition expansion, export; expression parser |
| Gate correctness | `tests/quantum_correctness/test_gates.py` | All 52 gate matrices equal Qiskit 2.5 **exactly incl. global phase** (`tests/golden/gates.json`, 3 parameter sets); every QASM `gate` definition we export equals the native gate; 12 random 40-gate circuits on up to 8 qubits vs an independent numpy simulator; identity/Pauli, H, Bell, GHZ (3/5/8), CNOT truth table, RX/RY/RZ at 5 angles, expressions, normalisation, χ² sampling test, seeds, partial measurement, invalid-circuit rejection |
| Cross-simulator | `tests/quantum_correctness/test_pennylane_crosscheck.py` | 10 random circuits (2–8 qubits, 27 gate types) — state fidelity vs PennyLane `default.qubit` = 1 ± 1e-10 |
| Examples | `tests/quantum_correctness/test_examples.py` | All 8 tutorials: exact probabilities equal theory, sampled counts within 5σ; QFT phases |
| API | `tests/integration/test_api.py` | Health, device/backends/gates, submit + retrieve, async polling, 8 invalid-input cases, 404, execution timeout, cancellation, 429 queue limit, 413 body limit, experiment CRUD and **persistence across an app restart**, invalid experiment rejected, QASM round trip + line-numbered error, examples endpoint, token auth (untrusted vs loopback), startup refusal without token, eject → 503 → reconnect, mock battery |
| SDK / PennyLane | `tests/integration/test_sdk.py` | SDK against a real uvicorn server: run, builder, QASM, errors, experiments, examples; PennyLane device (local engine) equals `default.qubit` for probs/expval incl. non-commuting observables; shots + remote mode; wire labels |
| QASM | `tests/integration/test_qasm_roundtrip.py` | 20 random 60-op circuits using every gate: export → import is bit-exact; Qiskit-style files with `gate` definitions import |
| Unit | `tests/unit/test_launcher_and_config.py` | Launcher opens once per connection, ignores ejected/foreign services; config precedence (env > file > profile), secrets never read from file; Pi profile requires a token; qubit bound; mock battery model; INA219 fallback |
| Frontend | `apps/studio/src/circuit/model.test.ts` | Grid placement, column shifting, parameter parsing, clbits, engine JSON round trip, visuals; API client errors and network failure |
| E2E | `tests/e2e/studio_e2e.py` | Real browser: dashboard, build Bell by click-to-place, run, histogram + probabilities + state vector, live validation error, QASM import, save experiment, tutorial run, eject overlay + reconnect, dark theme |

Shot-based assertions use statistical tolerances (χ² with p≈1e-4 threshold, or 5σ) instead of exact counts.

## Results on the development laptop (2026-10-09, WSL2 Ubuntu 24.04)

* `make -C engine test`: **169 checks, 0 failures**; self-test PASS
* `pytest tests`: **180 passed, 41 skipped** (skips = gates already in `qelib1.inc`, which need no exported
  `gate` definition)
* `npm test`: **10 passed**; `tsc`: no errors; production build 87 KB gzipped JS
* `studio_e2e.py` with Microsoft Edge (headless): **E2E PASSED**

## Requires the physical Raspberry Pi (not yet run)

* `install.sh` and `diagnose.sh` on Raspberry Pi OS 64-bit
* Engine build with `-mcpu=cortex-a72` and the full pytest suite on aarch64
* USB gadget enumeration on Windows/macOS/Linux hosts, DHCP, launcher auto-open
* VBUS isolation / back-feeding check with a USB power meter
* INA219 battery readings vs multimeter; charging detection; low-battery shutdown
* Speaker output of the eject voice and chimes; power-off after eject; data survival after reboot
* Pi benchmarks (`benchmarks/run_bench.py --extended`)
* Token enforcement over Wi-Fi from another machine
