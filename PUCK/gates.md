# Gate catalogue

All gates are hardcoded in `engine/src/gates.c` and match Qiskit's standard-gate matrices exactly
(including global phase), verified against `tests/golden/gates.json`. Qubit order is Qiskit's
little-endian convention: for `cx` on `[a, b]`, `a` is the control.

| name | qubits | params | category | QASM 2 | description |
|---|---:|---|---|---|---|
| `id` | 1 | – | pauli | `id` | Identity (no-op) |
| `x` | 1 | – | pauli | `x` | Pauli-X (bit flip, NOT) |
| `y` | 1 | – | pauli | `y` | Pauli-Y |
| `z` | 1 | – | pauli | `z` | Pauli-Z (phase flip) |
| `h` | 1 | – | clifford | `h` | Hadamard: creates equal superposition |
| `s` | 1 | – | clifford | `s` | Phase gate, sqrt(Z) |
| `sdg` | 1 | – | clifford | `sdg` | Inverse S gate |
| `sx` | 1 | – | clifford | `sx` | Square root of X (IBM native) |
| `sxdg` | 1 | – | clifford | `sxdg` | Inverse sqrt(X) |
| `t` | 1 | – | phase | `t` | T gate, fourth root of Z |
| `tdg` | 1 | – | phase | `tdg` | Inverse T gate |
| `p` | 1 | lambda | phase | `p` | Phase gate diag(1, e^{i lambda}) |
| `u1` | 1 | lambda | phase | `u1` | Legacy phase gate (same as p) |
| `rx` | 1 | theta | rotation | `rx` | Rotation about X axis |
| `ry` | 1 | theta | rotation | `ry` | Rotation about Y axis |
| `rz` | 1 | phi | rotation | `rz` | Rotation about Z axis |
| `r` | 1 | theta, phi | rotation | `r` (defined in file) | Rotation by theta about cos(phi)X + sin(phi)Y |
| `u` | 1 | theta, phi, lambda | universal | `u` | Generic single-qubit gate U(theta,phi,lambda) |
| `u2` | 1 | phi, lambda | universal | `u2` | Legacy U2(phi,lambda) = U(pi/2,phi,lambda) |
| `u3` | 1 | theta, phi, lambda | universal | `u3` | Legacy U3 (same as U) |
| `cx` | 2 | – | controlled | `cx` | Controlled-NOT (CNOT): control q0, target q1 |
| `cy` | 2 | – | controlled | `cy` | Controlled-Y |
| `cz` | 2 | – | controlled | `cz` | Controlled-Z (symmetric) |
| `ch` | 2 | – | controlled | `ch` | Controlled-Hadamard |
| `cs` | 2 | – | controlled | `cs` (defined in file) | Controlled-S |
| `csdg` | 2 | – | controlled | `csdg` (defined in file) | Controlled-S dagger |
| `csx` | 2 | – | controlled | `csx` | Controlled sqrt(X) |
| `cp` | 2 | lambda | controlled | `cp` | Controlled phase |
| `cu1` | 2 | lambda | controlled | `cu1` | Legacy controlled phase (same as cp) |
| `crx` | 2 | theta | controlled | `crx` | Controlled RX |
| `cry` | 2 | theta | controlled | `cry` | Controlled RY |
| `crz` | 2 | phi | controlled | `crz` | Controlled RZ |
| `cu` | 2 | theta, phi, lambda, gamma | controlled | `cu` | Controlled U with global phase gamma |
| `cu3` | 2 | theta, phi, lambda | controlled | `cu3` | Legacy controlled U3 |
| `swap` | 2 | – | two-qubit | `swap` | Swap two qubits |
| `iswap` | 2 | – | two-qubit | `iswap` (defined in file) | Swap with i phase on |01>,|10> |
| `dcx` | 2 | – | two-qubit | `dcx` (defined in file) | Double CNOT: cx(q0,q1) then cx(q1,q0) |
| `ecr` | 2 | – | two-qubit | `ecr` (defined in file) | Echoed cross-resonance (IBM native) |
| `rxx` | 2 | theta | interaction | `rxx` | Ising XX interaction exp(-i theta/2 XX) |
| `ryy` | 2 | theta | interaction | `ryy` (defined in file) | Ising YY interaction exp(-i theta/2 YY) |
| `rzz` | 2 | theta | interaction | `rzz` | Ising ZZ interaction exp(-i theta/2 ZZ) |
| `rzx` | 2 | theta | interaction | `rzx` (defined in file) | Cross-resonance exp(-i theta/2 Z(q0) X(q1)) |
| `xx_plus_yy` | 2 | theta, beta | interaction | `xx_plus_yy` (defined in file) | XX+YY interaction |
| `xx_minus_yy` | 2 | theta, beta | interaction | `xx_minus_yy` (defined in file) | XX-YY interaction |
| `ccx` | 3 | – | multi-controlled | `ccx` | Toffoli: controls q0,q1, target q2 |
| `ccz` | 3 | – | multi-controlled | `ccz` (defined in file) | Doubly-controlled Z |
| `cswap` | 3 | – | multi-controlled | `cswap` | Fredkin: control q0 swaps q1,q2 |
| `rccx` | 3 | – | multi-controlled | `rccx` | Relative-phase Toffoli (Margolus) |
| `c3x` | 4 | – | multi-controlled | `c3x` | 3-controlled X |
| `c3sx` | 4 | – | multi-controlled | `c3sqrtx` | 3-controlled sqrt(X) |
| `rc3x` | 4 | – | multi-controlled | `rc3x` | Relative-phase 3-controlled X |
| `c4x` | 5 | – | multi-controlled | `c4x` | 4-controlled X |

Aliases accepted on input: `cnot`→`cx`, `toffoli`→`ccx`, `fredkin`→`cswap`, `i`→`id`, `phase`→`p`, `cphase`→`cp`, `c3sqrtx`→`c3sx`, `rcccx`→`rc3x`, and the OpenQASM built-ins `U`, `CX`.
Also supported: `measure`, `reset` (mid-circuit), `barrier`.
