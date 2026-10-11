# Quanverge Quantum Puck — the whole project in questions and answers

A plain-language guide to everything in this repository: what was built, how it works, how to run it, how
the hardware and updates work, and what is still to do. Each answer points to the file or document with
more detail.

**Contents**
1. [The product](#1-the-product)
2. [Quantum basics used in the project](#2-quantum-basics-used-in-the-project)
3. [How the software is built](#3-how-the-software-is-built)
4. [The C engine](#4-the-c-engine)
5. [Quantum Studio (the web app)](#5-quantum-studio-the-web-app)
6. [Puck Pro and Notebook by Quanverge Labs](#6-puck-pro-and-notebook-by-quanverge-labs)
7. [Python, Qiskit, PennyLane and IBM code](#7-python-qiskit-pennylane-and-ibm-code)
8. [Running it on a laptop](#8-running-it-on-a-laptop)
9. [The hardware](#9-the-hardware)
10. [Plugging in, battery and eject](#10-plugging-in-battery-and-eject)
11. [Software updates](#11-software-updates)
12. [Security and privacy](#12-security-and-privacy)
13. [Testing and quality](#13-testing-and-quality)
14. [Performance](#14-performance)
15. [Licences, legal and commercial](#15-licences-legal-and-commercial)
16. [Limitations, open items and roadmap](#16-limitations-open-items-and-roadmap)
17. [Where to find things](#17-where-to-find-things)
18. [Troubleshooting](#18-troubleshooting)

---

## 1. The product

**Q: What is the Quantum Puck?**
A pocket-sized, battery-powered device that **simulates quantum computers**. You plug it into a laptop, a web app
called **Quantum Studio** opens, and you build and run quantum circuits — like IBM Quantum Composer, but running
entirely on the device, offline, with no cloud account.

**Q: Is it a real quantum computer?**
No. It is a **classical simulator**: an ordinary computer (a Raspberry Pi) calculating exactly what an ideal
quantum computer would do. This statement appears in the app, the API and the documentation, and must stay in all
marketing. It makes no claim of quantum hardware or "quantum advantage".

**Q: What are the two editions?**

| | Quantum Puck (Basic) | Quantum Puck Pro |
|---|---|---|
| Hardware | Raspberry Pi 4B 4 GB + Waveshare UPS HAT (B) | Raspberry Pi 5 8 GB + Geekworm X1200 |
| Qubits | 8 | 24 |
| Main use | Visual circuit building, tutorials | Everything in Basic + Python notebooks on the device |
| Programming | Studio, Python SDK from a laptop | Qiskit, PennyLane, NumPy in **Notebook by Quanverge Labs** |
| Folder | repository root | `quantum_puck_pro/` (reuses the Basic code) |

**Q: Who is it for?**
Students, teachers, universities and hobbyists learning quantum computing — people who want a private, offline,
hands-on device instead of a cloud account.

**Q: Why 8 qubits for the Basic edition?**
8 qubits is enough for every standard teaching example (Bell, GHZ, teleportation, Deutsch–Jozsa, Grover, QFT),
runs in milliseconds and needs only 4 KB of memory. The limit is a setting, not hard-coded: the engine supports up
to 24 qubits (28 in the Pro build).

---

## 2. Quantum basics used in the project

**Q: What is a qubit and a state vector?**
A qubit can be 0, 1 or a blend (superposition) of both. *n* qubits are described by a list of 2ⁿ complex numbers
called **amplitudes** — the state vector. 8 qubits → 256 amplitudes; 24 qubits → about 16.8 million. The simulator
stores this list and updates it gate by gate.

**Q: What is a gate?**
An operation on one or more qubits (e.g. **X** flips a qubit, **H** creates superposition, **CX/CNOT** entangles two
qubits). Mathematically each gate is a small matrix applied to the state vector.

**Q: What does "measurement" and "shots" mean?**
Measuring turns the blend into a definite 0/1 answer, at random, with probabilities given by the amplitudes.
Running the circuit many times ("shots", e.g. 1,024) gives a **histogram** of results. The Puck also reports the
**exact** probabilities, which a real quantum computer cannot.

**Q: What is "phase"?**
Each amplitude has a size (which gives the probability) and an angle (the phase). Phases don't change a single
measurement's probabilities but drive interference — they are why algorithms like QFT and Grover work. Studio's
**State vector** tab shows phases as coloured bars, dials and labels like `π/2`, in the style of IBM Composer.

**Q: Why are results written like `011`?**
The rightmost character is qubit 0 (Qiskit's convention, called little-endian). `011` means q0 = 1, q1 = 1, q2 = 0.
This matches IBM/Qiskit output exactly.

**Q: Which examples are included?**
Eight verified tutorials in `quantum/examples/` (docs: [examples.md](examples.md)): X gate, Hadamard, Bell state,
GHZ state, quantum teleportation, Deutsch–Jozsa, Grover search, and quantum Fourier transform. Pro adds eight
notebooks (Qiskit, IBM-style code, Aer comparison, PennyLane VQE, GHZ up to 22 qubits, NumPy).

---

## 3. How the software is built

**Q: What are the main parts?**

```
Browser (Quantum Studio)   Python SDK / PennyLane / notebooks
          │                        │
          ▼                        ▼
   FastAPI web service (Python) — REST API /api/v1, job queue, database, device control
          │  ctypes
          ▼
   libqpuck — the C simulation engine (gates, state vector, measurement, OpenQASM)
```

| Part | Language | Folder |
|---|---|---|
| Simulation engine | C11 | `engine/` |
| Web service / API | Python (FastAPI) | `services/api/` |
| Quantum Studio web app | React + TypeScript (Vite) | `apps/studio/` |
| Python SDK + PennyLane device | Python | `sdk/python/` |
| Puck Pro (notebooks, Qiskit backend) | Python + React | `quantum_puck_pro/` |
| Raspberry Pi installation, services, updater | Bash, systemd, Python | `device/` |
| Laptop auto-launcher | Python | `host-launcher/` |

Full description: [architecture.md](architecture.md).

**Q: Why C for the engine?**
Speed and a tiny footprint on a Raspberry Pi, no dependencies, and full ownership of the core technology. The
engine is a single library (`libqpuck.so`) that any language can call.

**Q: Why Python for the web service?**
Fast to develop and maintain, excellent web and testing tools, and the same language customers use for quantum
programming. The heavy maths happens in C, so Python's speed doesn't matter here.

**Q: How does a circuit get run?**
1. Studio sends the circuit as JSON to `POST /api/v1/jobs`.
2. The service checks it with the C validator (qubit indices, gate names, parameters, limits).
3. The job joins a bounded queue (max 16 waiting; more returns "queue full").
4. A worker thread calls the C engine, which can be cancelled or time out.
5. Results (counts, probabilities, state vector, depth, runtime, seed) are saved in SQLite and returned.

**Q: Where is data stored?**
In a SQLite database (`puck.sqlite3`) in the data folder (`/var/lib/quanverge` on a Pi, `var/` or `/tmp/...` on the
laptop). It holds jobs and saved experiments and survives restarts and power loss.

**Q: What is the circuit format?**
A simple JSON format documented in [circuit-schema.md](circuit-schema.md):
`{"num_qubits": 2, "gates": [{"name": "h", "qubits": [0]}, {"name": "cx", "qubits": [0, 1]}]}`.
Circuits can also be imported and exported as **OpenQASM 2.0** ([qasm-subset.md](qasm-subset.md)).

---

## 4. The C engine

**Q: Which gates are supported?**
All **52 standard gates of Qiskit/IBM** — Pauli, Clifford, phase, rotations, U gates, controlled gates, SWAP/iSWAP,
ECR, Ising interactions (RXX/RYY/RZZ/RZX), XX±YY, Toffoli, Fredkin, and multi-controlled X — plus measure, reset
and barrier. They are hardcoded in `engine/src/gates.c`. Full list: [gates.md](gates.md).

**Q: How do we know the gates are correct?**
Every gate matrix is compared with Qiskit's to 12 decimal places, **including global phase**
(`tests/golden/gates.json`). Random circuits are also cross-checked against an independent NumPy simulator and
PennyLane, and all 8 examples are checked against theory.

**Q: What else does the engine do?**
- Exact probabilities and seeded, reproducible shot sampling.
- Mid-circuit measurement and reset (switches to per-shot simulation).
- Circuit validation with clear error codes (e.g. `INVALID_QUBIT`, `UNKNOWN_GATE`).
- OpenQASM 2.0 parsing and export, including `gate` definitions and π expressions.
- Cancellation and time limits, output size caps for large circuits.
- An optional multithreaded build (OpenMP) for Puck Pro.

**Q: Can I use the engine without the web app?**
Yes: the `qpuck` command-line tool (`./engine/build/qpuck run circuit.json --shots 1000`), the Python binding
(`quanverge_puck.engine`), or the C API in `engine/include/qpuck.h`.

---

## 5. Quantum Studio (the web app)

**Q: What pages are there?**
- **Dashboard** — device status, battery, job statistics, getting-started tutorials, recent jobs.
- **Quantum Studio** — the circuit editor and results.
- **Notebook** — Pro only (see section 6).
- **Tutorials** — the 8 examples with explanations, expected vs measured results.
- **Experiments** — saved circuits and job history.
- **Device** — system information, battery, simulator limits, software updates, Eject, API token.

**Q: How do I build a circuit?**
Drag a gate from the colour-coded palette onto the grid, or click a gate then click a cell. For two- and
three-qubit gates, click the remaining qubits in the same column. Edit angles such as `pi/4` in the Inspector.
Shortcuts: ⌫ delete, ← → move, Esc cancel. The circuit is validated live; errors are shown on the gate.

**Q: What do the results show?**
Runtime, shots, depth, gate count and seed, plus four tabs: **Histogram** (measured vs exact), **Probabilities**,
**State vector** (phase-coloured bars, phase dials, π labels, colour-wheel legend) and **Raw JSON**.

**Q: Why are gates different colours?**
Each gate family has its own colour (Pauli, Clifford, Phase, Rotation, Universal, Controlled, Two-qubit,
Interaction, Multi-qubit); a key is shown under the circuit. The θ badge marks gates with an angle, dots show how
many qubits a gate uses.

**Q: Where do the branding, logo and colours come from?**
From www.quanvergelabs.com: the official logo, the teal/cyan palette and the Outfit + Playfair Display fonts. The
fonts are bundled so the app works offline. The app uses a light theme only.

**Q: Does the Pi need Node.js?**
No. The web app is built once on a laptop (`npm run build`) into `apps/studio/dist`, which the Pi serves as files.

---

## 6. Puck Pro and Notebook by Quanverge Labs

**Q: What does Pro add?**
**Notebook by Quanverge Labs** — Jupyter-style notebooks in the browser where Python runs **on the Puck itself**.
Qiskit, Qiskit Aer, PennyLane, NumPy, SciPy and matplotlib are installed. Pro also simulates up to 24 qubits with a
multithreaded engine.

**Q: Is it real Jupyter?**
The interface is our own (Quanverge design) but underneath it uses a real **IPython/Jupyter kernel**, so code
behaves exactly as in Jupyter: variables persist between cells, `%pip install` and other magics work, plots appear
inline, files are standard `.ipynb` and open in Jupyter or VS Code.

**Q: What can you do in a notebook?**
Code and Markdown cells, Shift/Ctrl/Alt+Enter, Run all, Stop (interrupt), Restart, autocompletion, inline plots,
coloured error messages, autosave, import/export `.ipynb`, a file manager with upload/download, and installing
extra Python packages when the Puck is online. Guide: `quantum_puck_pro/docs/notebook-guide.md`.

**Q: How are notebooks kept from crashing the device?**
At most 2 notebooks run at once, each with a memory limit (4–5 GB), CPU and process limits; idle notebooks stop
after 30 minutes. Kernels talk to the server over private Unix sockets (no network ports) and never see the API
token. Pro is designed as a single-user device.

**Q: How is Pro enabled?**
With the setting `edition = "pro"` (`PUCK_EDITION=pro`). The same web service, engine and app run in both editions;
the Notebook page appears only on Pro.

---

## 7. Python, Qiskit, PennyLane and IBM code

**Q: Can I control a Puck from Python on my laptop?**
Yes, with the Quanverge SDK (`sdk/python`):
```python
from quanverge_puck import QuantumPuckClient, Circuit
puck = QuantumPuckClient("http://localhost:8765")
print(puck.run(Circuit(2).h(0).cx(0, 1), shots=1000)["counts"])
```

**Q: How do Qiskit programs use the Puck?**
`from quanverge_pro import PuckBackend` gives a standard Qiskit backend, so `backend.run(qc)`, `transpile`, pass
managers and Qiskit's Sampler/Estimator all work on the Puck engine.

**Q: Can IBM Quantum tutorial code run on the Puck?**
Yes, by changing one import:
```python
# from qiskit_ibm_runtime import QiskitRuntimeService, SamplerV2 as Sampler, EstimatorV2 as Estimator
from quanverge_pro.runtime import QiskitRuntimeService, SamplerV2 as Sampler, EstimatorV2 as Estimator
```
This is Quanverge's own local implementation; it never contacts IBM and needs no account. Noise mitigation options
are accepted but ignored (the simulation is ideal). Details: `quantum_puck_pro/docs/ibm-migration.md`.

**Q: And PennyLane?**
`qml.device("quanverge.puck", wires=2)` (or `QuantumPuckDevice` in notebooks) runs PennyLane circuits on the Puck
engine, including gradients for variational algorithms.

**Q: Is Qiskit Aer included?**
Yes, on Pro, for comparison. The default simulator is always the Puck engine.

---

## 8. Running it on a laptop

**Q: How do I start it?**
In a WSL (Ubuntu) terminal inside the project folder:

| Edition | Command | Address |
|---|---|---|
| Basic | `./scripts/run-laptop.sh` | http://localhost:8765 |
| Pro | `./quantum_puck_pro/scripts/run-pro-laptop.sh` | http://localhost:8766/#/notebook |

Both can run at the same time. The first start builds the engine and installs Python packages.

**Q: Why port 8765 and not 8080?**
Port 8080 is used by Apache/XAMPP on this laptop, so the Puck uses 8765 (Pro: 8766). Change with `PUCK_PORT`.

**Q: What is the "virtual Puck"?**
The laptop version: identical software, but the battery is simulated, Eject plays the announcement in the browser
and offers "Reconnect", and software updates are verified in a sandbox instead of replacing the running code.

**Q: How do I stop it?**
Press Ctrl+C in the terminal that started it.

---

## 9. The hardware

**Q: What do I need to build a Basic Puck?**
Raspberry Pi 4B 4 GB, Waveshare UPS HAT (B), 2 × 18650 cells, a PAM8302 amplifier + small speaker, a
VBUS-blocking USB-C adapter + data cable, a microSD card and a case — about $55–70 on top of the Pi.

**Q: And a Pro?**
Raspberry Pi 5 8 GB, Active Cooler, Geekworm X1200, 2 × 18650 cells, 27 W USB-C PD charger, 64 GB microSD, a USB
audio adapter (the Pi 5 has no headphone jack) — about $140–160.

**Q: How do I build and install it?**
Follow [BUILD.md](BUILD.md): flash Raspberry Pi OS 64-bit, assemble, copy the project, run
`sudo ./device/scripts/install.sh` (Pro: `sudo ./quantum_puck_pro/device/install-pro.sh`), reboot, run
`diagnose.sh`, then the acceptance checklist.

**Q: Has it been tested on a real Raspberry Pi?**
**Not yet.** All software is tested on the laptop. The Pi steps (USB networking, battery gauge, speaker, power-off)
were written from vendor documentation and must be confirmed with the checklist in BUILD.md Part 4 before the
device is relied on or sold.

**Q: Why the "VBUS-blocking" adapter?**
The battery board powers the Pi, and a normal USB-C cable from a laptop would also push power into the same wire.
The adapter passes data only. A commercial version should build this protection inside the case.

---

## 10. Plugging in, battery and eject

**Q: How does "plug in and the app opens" work?**
Over USB-C the Pi pretends to be a USB network adapter (address `192.168.7.1`). A small **host launcher**, installed
once per laptop (`host-launcher/install-windows.ps1`), notices the Puck and opens Studio. Operating systems do not
let USB devices start programs by themselves, which is why the launcher is needed.

**Q: Can it be used over Wi-Fi?**
Yes, at `http://quanverge-puck.local` (Pro: `quanverge-puck-pro.local`). Over Wi-Fi an API token is required;
over the USB cable it is not.

**Q: How does the battery reading work?**
Basic reads the INA219 gauge on the UPS HAT (B); Pro reads the X1200's MAX17040 gauge and its power-loss signal.
Battery %, charging state and voltage appear on the Device page. Below 15 % the Puck warns; below 8 % it shuts down
safely by itself.

**Q: How do I charge it?**
Basic: through the UPS HAT's own DC charger. Pro: through the X1200's USB-C PD port. Plugging into a laptop does not
charge either edition.

**Q: What does Eject do?**
It stops the job queue, saves the database to disk, says **"You are safe to remove now"** through the Puck's
speaker and powers off. On the laptop the message plays in the browser.

---

## 11. Software updates

**Q: How do devices get new software?**
Over Wi-Fi, every 6 hours, the Puck checks Quanverge's update server (website first, GitHub Releases as backup).
When a newer version exists, Studio shows **"Update available"** and the user installs it with one click on the
Device page. Details: [UPDATES.md](UPDATES.md).

**Q: What stops a fake or broken update?**
Every release is **signed** with Quanverge's private Ed25519 key; each Puck holds the matching public key and
rejects anything else. A separate system service re-checks the signature and file checksum, installs the new
version next to the old one, tests it, switches over, and **rolls back automatically** if it does not start.
User data is never touched.

**Q: When is installing blocked?**
While jobs or notebooks are running, or when the battery is under 30 % and not charging.

**Q: How do I publish an update?**
Bump `VERSION`, run `npm run build`, then `python3 scripts/release/make_release.py --notes "…"`, and upload the
three files to the website folder and a GitHub release. Step by step: [UPDATES.md](UPDATES.md).

**Q: Where is the signing key and why does it matter so much?**
The private key is at `~/.quanverge/release-keys/update-signing.key.pem` (in WSL) and must be backed up in two safe
places. Losing it means sold devices can no longer be updated over the air; leaking it means someone else could
sign updates. It must never go into Git or a release.

**Q: Is the update server set up?**
Not yet. `updates.quanvergelabs.com` and the public `quanverge-labs/quantum-puck-releases` repository still need to
be created; until then devices report "no update has been published yet".

---

## 12. Security and privacy

**Q: Does the Puck send any data to the internet?**
No telemetry. It only contacts the update servers (and package servers if the user installs Python packages on
Pro). Circuits, results and notebooks stay on the device.

**Q: Who can access a Puck?**
On the laptop: only the laptop itself (localhost). On a Pi: the laptop connected over USB, plus anyone on the
Wi-Fi network **who has the API token**. The token is generated at installation in `/etc/quanverge/puck.env`.

**Q: What other protections exist?**
- Every circuit is validated before running; sizes, depth, shots and run time are limited.
- Request size limits, a bounded job queue, safe error messages without internal details.
- The service runs as an unprivileged user with a read-only system and no extra privileges.
- Notebook file access is confined to the workspace (no `..` or absolute paths); uploaded HTML is sanitised.
- Updates are signed and verified as described above.

**Q: Should a Puck be exposed to the internet?**
No. Use it over USB or on trusted networks only; never port-forward it.

---

## 13. Testing and quality

**Q: How is the project tested?**

| Suite | What it checks | Count |
|---|---|---|
| C engine (`make -C engine test`) | gates, kernels, errors, QASM, sampling, cancellation | 169 checks |
| Base Python (`pytest tests`) | gate correctness vs Qiskit, NumPy & PennyLane, API, SDK, QASM, config, updates | 191 tests |
| Pro (`pytest quantum_puck_pro/tests`) | Qiskit backend vs Qiskit/Aer, IBM-style code, notebook kernels, all samples, limits | 43 tests |
| Web app (`npm test`) | circuit model and API client | 10 tests |
| Browser end-to-end | Studio and Notebook driven in Microsoft Edge | 2 scripts |

Shot-based checks use statistical tolerances rather than exact counts. Details: [testing.md](testing.md).

**Q: What has not been tested?**
Anything requiring the physical hardware: Pi installation, USB networking, battery readings, the speaker,
power-off and Pi performance.

---

## 14. Performance

**Q: How fast is it?**
On the development laptop, an 8-qubit circuit with 460 gates runs in about 1–2 ms. Time doubles roughly with each
added qubit: about 0.1 s at 16 qubits and a few seconds at 20 qubits. Memory is tiny at 8 qubits and about
250 MB at 22, 920 MB at 24. Details: [benchmarks.md](benchmarks.md) and `quantum_puck_pro/docs/benchmarks.md`.

**Q: Does multithreading help?**
Moderately: 1.3–1.4× with 2–4 threads at 20 qubits on the laptop, because the work is limited by memory speed.
Pro uses one thread per physical core (maximum 4). Raspberry Pi numbers still have to be measured.

**Q: How can it get faster later?**
Gate fusion (combining consecutive single-qubit gates), cache blocking, and an optional single-precision mode.

---

## 15. Licences, legal and commercial

**Q: Are Qiskit and PennyLane free to use in a commercial product?**
Yes. Both are Apache-2.0: commercial use, bundling and selling are allowed, and our own code can stay private.
Obligations: include their licence texts (a `THIRD_PARTY_NOTICES` file and an About page — still to do), and do not
use their trademarks or logos to imply endorsement. Other libraries are MIT/BSD/Apache; the fonts are OFL.

**Q: Any legal risks in the current setup?**
None known for the software as used, but: add the licence notices, have a lawyer review them and the IBM-style
class names in the compatibility layer, check that "Quantum Puck" can be trademarked, and keep the "classical
simulator" statement. This is general guidance, not legal advice.

**Q: What else is needed to sell it?**
Hardware validation; a customer-friendly design (USB-C charging, built-in power isolation, power button, status
LED, proper enclosure); certification (CE/FCC, BIS in India); certified lithium cells (IEC 62133, UN38.3); a
signed laptop launcher installer; a first-time setup screen with a per-device password; a shippable OS image;
user documentation; pricing, warranty and support.

---

## 16. Limitations, open items and roadmap

**Q: What are the current limitations?**
- Raspberry Pi hardware support is unverified.
- No classical control flow in circuits (`if` on measurement results) — use the deferred-measurement form.
- No noise models (results are ideal).
- OpenQASM 2.0 subset only, no OpenQASM 3.
- Very large results are capped (top 4,096 outcomes; state vectors above 14 qubits are not returned).
- Pro notebooks run with the service's permissions (single-user device).

**Q: How does it grow from 8 to 20+ qubits?**
Change the `max_qubits` setting after benchmarking on the device; the engine already supports it. See
[roadmap-20-qubits.md](roadmap-20-qubits.md).

**Q: What are the next software steps?**
Commit to Git with automatic testing; licence notices and About page; first-time setup screen and unique device
names; a Windows launcher installer; then noise simulation, classroom features and more lessons.

**Q: What are the next steps for the founder?**
Back up the signing key, create the GitHub repositories, try both editions on the laptop, order prototype parts,
build and test, set up the update server, then trademark, certification quotes and pilot users. The full ordered
list is in the project discussion and summarised in [BUILD.md](BUILD.md) and [UPDATES.md](UPDATES.md).

---

## 17. Where to find things

| I want to… | Look at |
|---|---|
| Understand the overall design | [architecture.md](architecture.md) |
| Use the HTTP API | [api.md](api.md), [openapi.json](openapi.json), `/api/docs` on a running Puck |
| Write circuits as JSON / QASM | [circuit-schema.md](circuit-schema.md), [qasm-subset.md](qasm-subset.md) |
| See every gate | [gates.md](gates.md) |
| See the tutorial results | [examples.md](examples.md) |
| Build a device | [BUILD.md](BUILD.md), [hardware.md](hardware.md), `quantum_puck_pro/docs/hardware-pi5.md` |
| Publish an update | [UPDATES.md](UPDATES.md) |
| Run or read the tests | [testing.md](testing.md) |
| Check performance | [benchmarks.md](benchmarks.md) |
| Use notebooks / IBM code | `quantum_puck_pro/README.md`, `quantum_puck_pro/docs/` |
| Change the engine | `engine/src/` (gates in `gates.c`, simulation in `sim.c`, QASM in `qasm.c`) |
| Change the web app | `apps/studio/src/` (pages, circuit editor, visualisations) |
| Change the API | `services/api/app/` (routes, jobs, store, device, updates) |

---

## 18. Troubleshooting

**Q: The browser says it cannot reach the Puck.**
Make sure the start script is still running in its terminal and use the right address (8765 Basic, 8766 Pro). On a
Pi, check `systemctl status quanverge-api` and the USB cable or Wi-Fi.

**Q: The page shows "You are safe to remove now" and nothing works.**
The Puck was ejected. On the laptop click **Reconnect**; on a Pi, switch it off and on again.

**Q: Studio looks old after an update.**
Press Ctrl+Shift+R to reload without the browser cache.

**Q: A circuit won't run.**
Read the red message under the circuit: it names the problem (e.g. a missing parameter or a qubit out of range)
and the gate is outlined in red.

**Q: A notebook kernel won't start (Pro).**
Restart the Pro server; on a Pi check `journalctl -u quanverge-api -e`.

**Q: Updates say "no update has been published yet".**
The update server and releases repository have not been set up yet (see section 11).

**Q: Where are the logs?**
Laptop: the terminal running the start script. Pi: `journalctl -u quanverge-api -f`, and
`sudo /opt/quanverge/device/scripts/diagnose.sh` for a full health report.
