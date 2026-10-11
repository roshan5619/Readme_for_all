# Circuit JSON schema (v1)

The single interchange format used by the engine, the API, the SDK and the Studio.

```json
{
  "num_qubits": 3,
  "num_clbits": 3,
  "gates": [
    {"name": "h", "qubits": [0]},
    {"name": "cx", "qubits": [0, 1]},
    {"name": "rz", "qubits": [2], "params": ["pi/4"]},
    {"name": "cu", "qubits": [0, 2], "params": [0.1, 0.2, 0.3, 0.0]},
    {"name": "barrier", "qubits": [0, 1, 2]},
    {"name": "measure", "qubits": [2], "clbits": [0]},
    {"name": "reset", "qubits": [2]}
  ]
}
```

| Field | Type | Rules |
|---|---|---|
| `num_qubits` | int | 1 … device `max_qubits` (8 by default; engine ceiling 24) |
| `num_clbits` | int, optional | 0 … 64. Default: `num_qubits` (or highest measured clbit + 1) |
| `gates` | array | Ordered operations, applied first to last. Alias key: `operations` |
| `measure_all` | bool, optional | Only meaningful without `measure` ops; `false` disables implicit measurement |

Operation fields:

| Field | Type | Rules |
|---|---|---|
| `name` | string | A gate from [gates.md](gates.md), an alias, or `measure` / `reset` / `barrier` |
| `qubits` | int[] | Exactly the gate's arity, distinct, each `0 ≤ q < num_qubits`. Order matters: for controlled gates the leading qubits are controls (`cx [c, t]`, `ccx [c1, c2, t]`, `cswap [c, a, b]`) |
| `params` | (number \| string)[] | Exactly the gate's parameter count, radians. Strings are expressions: numbers, `pi`, `+ - * / ^`, parentheses, unary minus, `sin cos tan exp ln sqrt` — e.g. `"pi/2"`, `"-3*pi/4"`. Must be finite with \|value\| ≤ 1e6 |
| `clbits` | int[1] | `measure` only; default = the measured qubit index |

## Semantics

* Qubit ordering is Qiskit's little-endian convention; result bitstrings print `c[n-1]…c[0]`.
* **Implicit measurement**: if the circuit has no `measure`, all qubits are measured at the end into clbits with
  the same index.
* **Terminal vs. mid-circuit measurement**: if no gate or reset touches a qubit after it is measured, results come
  from one exact simulation (`"method": "statevector"`, exact probabilities). Otherwise each shot is simulated
  with state collapse (`"statevector_per_shot"`, probabilities estimated from shots, no state vector).
* `barrier` has no effect on the state; it only separates columns/depth.
* Classical control (`if`) is **not** part of the MVP. Use the deferred-measurement form (controlled gates), as in
  the teleportation example.

## Validation (always before execution)

`qp_validate` rejects with a specific code and `op_index`: unknown gate, wrong qubit/parameter count, out-of-range
or duplicate qubit, non-finite/out-of-range parameter, clbit out of range, too many qubits/ops, depth above the
limit, shots above the limit. See the error table in [api.md](api.md).
