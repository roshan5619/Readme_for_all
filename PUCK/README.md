# Quanverge Labs — Quantum Puck

A pocket-sized, local-first **quantum circuit simulator**: a C state-vector engine with the full IBM/Qiskit
standard gate set, a REST API, the **Quantum Studio** web app, a Python SDK and a PennyLane device. It runs on a
laptop today ("virtual Puck") and is packaged for a battery-powered **Raspberry Pi 4B** that plugs into a laptop
over USB-C.

> Quantum Puck is a *classical* simulator of quantum circuits running on ordinary CPUs. It is not a quantum
> processor and makes no claim of quantum advantage.

**MVP scope:** 8 qubits · 52 standard gates (+ measure, reset, barrier) · exact probabilities, seeded shot sampling,
state vectors · mid-circuit measurement · OpenQASM 2.0 import/export (documented subset) · experiment history ·
offline operation · 20-qubit roadmap by configuration.

## Quick start — laptop (virtual Puck)

Requirements: WSL Ubuntu, Linux or macOS with `gcc`/`make` and Python ≥ 3.11 (3.12 tested). Node.js is only needed
to rebuild the UI (a prebuilt `apps/studio/dist` is included).

```bash
./scripts/run-laptop.sh
```

This builds the C engine, creates a Python environment in `~/.venvs/quanverge-puck`, installs the API
dependencies, and serves **Quantum Studio at http://localhost:8765** (API docs at `/api/docs`), then opens your
browser. On Windows run it from the WSL terminal inside this folder:

```powershell
wsl -e bash -lc "cd /mnt/c/Users/<you>/Documents/Quanverge_Labs/Quantum_Puck && ./scripts/run-laptop.sh"
```

The laptop profile binds to `127.0.0.1` only, simulates the battery, and plays the eject announcement
("You are safe to remove now") through the laptop speakers.

> Port 8765 is used because 8080 is commonly taken (e.g. by XAMPP/Apache). Override with `PUCK_PORT=9000`.

### Use it

1. **Quantum Studio** → pick a gate from the palette (drag it onto the grid, or click it and click a cell; for
   multi-qubit gates click the remaining qubits in the same column). Edit parameters such as `pi/4` in the
   Inspector; ⌫ deletes, ← → move, Esc cancels.
2. **Run** → histogram (sampled vs exact), probability table, state vector with phases, depth, gate count, runtime.
3. **Save** an experiment, **Import/Export** JSON or OpenQASM 2.0, browse **Tutorials** (8 verified examples),
   check **Device** (battery, system, limits, Eject).

### Python SDK

```bash
~/.venvs/quanverge-puck/bin/pip install -e "sdk/python[pennylane]"
```
```python
from quanverge_puck import QuantumPuckClient, Circuit
puck = QuantumPuckClient("http://localhost:8765")
print(puck.run(Circuit(2).h(0).cx(0, 1), shots=1000, seed=7)["counts"])   # {'00': ~500, '11': ~500}

import pennylane as qml
dev = qml.device("quanverge.puck", wires=2)        # in-process C engine; add url=... for a remote Puck
```

### Engine CLI

```bash
make -C engine && ./engine/build/qpuck selftest
echo '{"num_qubits":2,"gates":[{"name":"h","qubits":[0]},{"name":"cx","qubits":[0,1]}]}' | ./engine/build/qpuck run - --shots 1000 --seed 1
```

## Quantum Puck Pro — Python notebooks on the device

**Puck Pro** adds *Notebook by Quanverge Labs*: Jupyter-compatible notebooks where Python (Qiskit, Qiskit Aer,
PennyLane, NumPy, matplotlib) runs on the Puck itself, a Qiskit backend for the Puck engine, and an IBM
Quantum–style runtime so IBM tutorial code runs after changing one import. Target hardware: Raspberry Pi 5 8 GB +
Geekworm X1200. Laptop: `./quantum_puck_pro/scripts/run-pro-laptop.sh` → http://localhost:8766/#/notebook.
See [quantum_puck_pro/README.md](quantum_puck_pro/README.md).

## Raspberry Pi (Quantum Puck hardware)

Pi 4B 4 GB + Waveshare UPS HAT (B) + small speaker, ≈ $55–70 of new parts. Full guide, wiring and power caveats:
[docs/hardware.md](docs/hardware.md).

```bash
sudo ./device/scripts/install.sh --hostname quanverge-puck && sudo reboot
sudo /opt/quanverge/device/scripts/diagnose.sh --sound
```

Then on each laptop install the launcher once (`host-launcher/install-windows.ps1` or `install-unix.sh`): plug the
Puck in and Studio opens at `http://192.168.7.1`. **The Pi steps have not yet been executed on real hardware** —
see the acceptance checklist in the hardware guide.

## Testing

```bash
make -C engine test                                   # 169 C checks
~/.venvs/quanverge-puck/bin/pip install -r requirements-dev.txt
~/.venvs/quanverge-puck/bin/python -m pytest tests    # correctness, API, SDK, PennyLane, QASM, config
cd apps/studio && npm install && npm test             # UI unit tests
python tests/e2e/studio_e2e.py                        # real-browser E2E (Playwright + Edge/Chrome)
```

Correctness is checked against Qiskit-generated golden matrices (exact, including global phase), an independent
numpy simulator and PennyLane `default.qubit`. Details: [docs/testing.md](docs/testing.md).

## Repository layout

```
engine/            C11 simulator: gates.c (52 gates), sim.c (kernels, sampling), circuit.c (validation),
                   qasm.c (OpenQASM 2), api.c (JSON API), cli.c, tests/
services/api/app/  FastAPI service: routes/, schemas, jobs (bounded queue), store (SQLite), device, security
apps/studio/       React + TypeScript + Vite UI (dist/ is prebuilt)
sdk/python/        quanverge_puck: HTTP client, Circuit builder, ctypes engine binding, PennyLane device
quantum/examples/  8 tutorial circuits (JSON) + build_examples.py
device/            Pi installer, diagnostics, USB gadget, systemd units, configuration, sounds
host-launcher/     auto-open Studio when the Puck is plugged in (Windows/macOS/Linux)
benchmarks/        run_bench.py, REPORT_TEMPLATE.md, results/
tests/             golden/, quantum_correctness/, integration/, unit/, e2e/
docs/              architecture, api, circuit schema, QASM subset, gates, examples, hardware, benchmarks,
                   testing, roadmap
scripts/           run-laptop.sh, gen_docs.py
```

## Documentation

**New to the project? Read [docs/PROJECT_QA.md](docs/PROJECT_QA.md)** — everything explained in questions and answers.

**Building a device? Start with [docs/BUILD.md](docs/BUILD.md)** — parts, assembly, installation and checks for the Puck (Pi 4B) and Puck Pro (Pi 5).

[Architecture](docs/architecture.md) · [API](docs/api.md) · [Circuit schema](docs/circuit-schema.md) ·
[OpenQASM subset](docs/qasm-subset.md) · [Gates](docs/gates.md) · [Examples with verified results](docs/examples.md) ·
[Hardware & Pi setup](docs/hardware.md) · [Benchmarks](docs/benchmarks.md) · [Testing](docs/testing.md) ·
[20-qubit roadmap](docs/roadmap-20-qubits.md)

## Known limitations

* **Raspberry Pi support is unverified** until the hardware checklist passes; USB gadget mode, VBUS isolation,
  INA219 calibration and audio routing are written from documentation, not yet tested on the device.
* The UPS HAT (B) charges from its own DC input, not from the laptop's USB port.
* "Plug in → app opens" needs the small host launcher installed once per laptop (operating systems block USB autorun).
* No classical control flow (`if`) — use the deferred-measurement form. No noise models.
* OpenQASM 2.0 subset only (see docs); no OpenQASM 3.
* Simulation is single-threaded; 8 qubits run in milliseconds, 20 qubits in seconds on a laptop (Pi slower).
* Results with very many outcomes are capped (top 4,096 probabilities; state vectors above 14 qubits are not
  returned).
* The logo is a text-and-glyph placeholder pending an approved brand asset.
