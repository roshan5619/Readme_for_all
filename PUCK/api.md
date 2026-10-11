# HTTP API (v1)

Base URL: `http://localhost:8765` (laptop) · `http://192.168.7.1` (Puck over USB-C) · `http://<hostname>.local`
(Puck over Wi-Fi/Ethernet, token required). Interactive docs: `/api/docs`. Machine-readable schema:
[`openapi.json`](openapi.json) (regenerate with `python scripts/gen_docs.py`).

## Authentication

On the Pi, requests from outside loopback and the USB link must send `Authorization: Bearer <PUCK_API_TOKEN>`
(or `X-Puck-Token`). `GET /api/v1/health` is always open. Missing/invalid token → `401 UNAUTHORIZED`.

## Errors

Every error has the same shape:

```json
{"error": {"code": "INVALID_QUBIT", "message": "operation 1 (cx): qubit index 3 out of range 0..1", "op_index": 1}}
```

| HTTP | code(s) |
|---|---|
| 401 | `UNAUTHORIZED` |
| 404 | `NOT_FOUND` |
| 408 | `TIMEOUT` (validate-time only; execution timeouts are reported on the job) |
| 413 | `PAYLOAD_TOO_LARGE` |
| 422 | `INVALID_REQUEST` (schema), `INVALID_JSON`, `INVALID_CIRCUIT`, `TOO_MANY_QUBITS`, `UNKNOWN_GATE`, `INVALID_QUBIT`, `DUPLICATE_QUBIT`, `WRONG_QUBIT_COUNT`, `WRONG_PARAM_COUNT`, `INVALID_PARAM`, `INVALID_CLBIT`, `TOO_MANY_GATES`, `CIRCUIT_TOO_DEEP`, `INVALID_SHOTS`, `UNKNOWN_BACKEND`, `QASM_SYNTAX` / `QASM_UNSUPPORTED` (with `line`) |
| 429 | `QUEUE_FULL` (has `Retry-After`) |
| 503 | `DEVICE_EJECTED` |

Job-level failures (`TIMEOUT`, `CANCELLED`, `OUT_OF_MEMORY`, `INTERRUPTED`) appear in the job's `error` field.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/v1/health` | Liveness, version, device state, queue depth |
| GET | `/api/v1/device` | Device, engine, limits, system metrics, battery, eject state |
| GET | `/api/v1/device/battery` | Battery reading |
| PUT | `/api/v1/device/battery/mock` | Laptop only: set simulated `charging` / `percent` |
| POST | `/api/v1/device/eject` | Safe removal (stop queue, flush, announce; Pi powers off) |
| POST | `/api/v1/device/reconnect` | Laptop only: simulate re-plugging |
| GET | `/api/v1/backends` | Available simulator backend(s) and limits |
| GET | `/api/v1/gates` | Gate catalogue |
| POST | `/api/v1/jobs?wait=s` | Submit a job (optionally wait up to 30 s) → 202 |
| GET | `/api/v1/jobs?limit&offset` | Recent jobs |
| GET | `/api/v1/jobs/{id}?wait=s` | Job status/result (long-poll) |
| POST | `/api/v1/jobs/{id}/cancel` | Cancel queued or running job |
| GET | `/api/v1/experiments` | Saved experiments |
| POST | `/api/v1/experiments` | Save (validated circuit + optional `last_job_id`) → 201 |
| GET/PUT/DELETE | `/api/v1/experiments/{id}` | Read / update / delete |
| GET | `/api/v1/metrics` | Job counters, queue, average runtime, CPU/memory/temperature |
| POST | `/api/v1/circuits/validate` | Validate without running (depth, gate count, memory estimate) |
| POST | `/api/v1/qasm/import` | OpenQASM 2.0 → circuit JSON |
| POST | `/api/v1/qasm/export` | circuit JSON → OpenQASM 2.0 |
| GET | `/api/v1/examples`, `/api/v1/examples/{id}` | Bundled tutorials |
| GET | `/api/v1/updates` | Software update status (installed version, available update, progress, last result) |
| POST | `/api/v1/updates/check` | Check the update servers now (Wi-Fi) |
| POST | `/api/v1/updates/install` | Download, verify and install the available update (409 `BUSY` / `NO_UPDATE` / `INCOMPATIBLE`) |

## Submitting a job

```bash
curl -s -X POST "http://localhost:8765/api/v1/jobs?wait=10" -H 'Content-Type: application/json' -d '{
  "circuit": {"num_qubits": 2, "gates": [{"name": "h", "qubits": [0]}, {"name": "cx", "qubits": [0, 1]}]},
  "shots": 1000, "seed": 42, "backend": "qpuck.statevector", "return_statevector": true, "name": "bell"
}'
```

Request fields: `circuit` (required, see [circuit-schema.md](circuit-schema.md)), `shots` (0–`max_shots`,
default 1024), `seed` (optional, 0…2^53−1), `backend` (`qpuck.statevector`), `return_statevector` (bool),
`name`, `experiment_id`.

Response (abridged):

```json
{
  "id": "8cc3bdfb…", "status": "completed", "backend": "qpuck.statevector", "shots": 1000, "num_qubits": 2,
  "created_at": 1791542680.20, "started_at": 1791542680.20, "finished_at": 1791542680.21,
  "result": {
    "engine": "qpuck-c 0.1.0", "seed": 42, "depth": 3, "gate_count": 2, "op_counts": {"h": 1, "cx": 1, "measure": 2},
    "method": "statevector", "probabilities_exact": true,
    "probabilities": {"00": 0.5, "11": 0.5}, "num_outcomes": 2, "probabilities_truncated": false,
    "counts": {"00": 506, "11": 494},
    "statevector": [[0.7071067811865476, 0], [0, 0], [0, 0], [0.7071067811865476, 0]],
    "runtime_ms": 0.052, "simulation_ms": 0.051, "memory_bytes": 96
  }
}
```

Bitstrings are classical registers written `c[n-1]…c[0]` (qubit/clbit 0 is the **rightmost** character), as in
Qiskit. Without explicit `measure` operations every qubit is measured into the clbit with the same index.

## Python SDK

```python
from quanverge_puck import QuantumPuckClient, Circuit
puck = QuantumPuckClient("http://localhost:8765")
res = puck.run(Circuit(2).h(0).cx(0, 1), shots=1000, seed=1)
```

See [`sdk/python/README.md`](../sdk/python/README.md) for the PennyLane device.
