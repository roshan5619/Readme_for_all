# OpenQASM 2.0 support (documented subset)

Quantum Puck imports and exports **OpenQASM 2.0**. It does not claim full OpenQASM compatibility; the supported
subset is listed here and anything else is rejected with an error that names the line.

## Import — supported

* Header `OPENQASM 2.0;` (required) and `include "qelib1.inc";` (the only include accepted).
* `qreg` / `creg` — several registers allowed; they are concatenated in declaration order.
* Every gate in [gates.md](gates.md) by its QASM name (e.g. `c3sqrtx` for `c3sx`, `rcccx` for `rc3x`) plus the
  built-ins `U(θ,φ,λ)` and `CX`.
* Register broadcasting: `h q;`, `cx a, b;` (equal sizes), `measure q -> c;`.
* `measure`, `reset`, `barrier`.
* `gate` definitions: expanded inline (nested up to 16 levels). A definition for a name the engine already knows
  (as emitted by Qiskit for `ecr`, `iswap`, …) is ignored and the exact built-in is used.
* Parameter expressions: numbers, `pi`, `+ - * / ^`, unary minus, parentheses, `sin cos tan exp ln sqrt`.
* `//` and `/* */` comments.

## Import — rejected (error code)

| Construct | Code |
|---|---|
| `OPENQASM 3.x` | `QASM_UNSUPPORTED` |
| `if (c == n) …` classical control | `QASM_UNSUPPORTED` |
| `opaque` declarations | `QASM_UNSUPPORTED` |
| includes other than `qelib1.inc` | `QASM_UNSUPPORTED` |
| unknown gate / undefined identifier | `UNKNOWN_GATE` / `QASM_SYNTAX` |
| index out of range | `INVALID_QUBIT` |
| more qubits than the engine ceiling (24) | `TOO_MANY_QUBITS` |

Importing a circuit wider than the device limit (8) succeeds with a warning; running it is rejected.

## Export

* Header + `include "qelib1.inc";`, `qreg q[n];`, `creg c[m];` when measurements exist.
* Gates in `qelib1.inc` are written by name. Standard gates outside `qelib1.inc` (`r`, `cs`, `csdg`, `ccz`, `iswap`,
  `dcx`, `rzx`, `ecr`, `ryy`, `xx_plus_yy`, `xx_minus_yy`) are preceded by a `gate` definition using the same
  decompositions as Qiskit's exporter, so the file loads in other tools. Each definition is verified in
  `tests/quantum_correctness/test_gates.py::test_qasm_definitions_are_correct` (equal up to global phase).
* Angles are printed as exact multiples of π when that round-trips bit-for-bit (`pi/2`, `-3*pi/4`), otherwise
  with 17 significant digits — export → import reproduces the same parameters exactly.
