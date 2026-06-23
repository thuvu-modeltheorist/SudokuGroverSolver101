# Quantum Grover Solver for 4×4 Sudoku

This project implements a Grover search for a **4×4 Sudoku**.

Each blank Sudoku cell is encoded with two qubits:

| Digit | Encoding |
|---|---|
| 1 | `00` |
| 2 | `01` |
| 3 | `10` |
| 4 | `11` |

We assumes that all provided cells are already valid: no two given cells repeat a value in the same row, column, or 2×2 box.

The circuit allocates search qubits only for blank cells. Fixed clues are treated as classical constants.

## Main idea
For each blank cell, the oracle checks:

-The value is compatible with all fixed neighbors in its row, column, and box.

The oracle phase-marks assignments satisfying all reduced constraints. Grover's diffuser then amplifies the marked assignments.

Here we have the oracle to compare two cells.

<img width="467" height="230" alt="image" src="https://github.com/user-attachments/assets/1dbfcee2-0cf0-4192-8cd7-3cd75780c35f" />

The resulting clause value is:

- `1` when the cells differ;
- `0` when the cells are equal.
  
## Reduced qubit count

Let:

- `u` be the number of blank cells;
- `m` be the number of pairs of blank cells that share a row, column, or box.

Every cell has seven distinct Sudoku neighbors: cells sharing its row, column, or 2×2 box.

Two empty cells therefore contribute 7*2=14 neighbor incidences, so we need at most 14 clauses here.

Each empty-empty pair was counted twice in those 14 incidences but needs only one clause. In our problem, the two empty cells are in different rows and columns so m=0. But if m>0, then the number of clauses is 7u-m.

Thus, the reduced four-blank construction fits within a (32+14+2+1=49)-qubit limit.

## Search-space size

With `u` blank cells, each having four possible values,

```text
N = 4^u.
```

For two blank cells,

```text
N = 4^2 = 16.
```

If there is one valid completion, three Grover iterations give a high success probability.

## Diffuser

The diffuser acts only on the `2u` search qubits belonging to blank cells.

```python
from qiskit import QuantumCircuit
from qiskit.circuit.library import MCXGate

def diffuser(number_of_qubits):
    circuit = QuantumCircuit(
        number_of_qubits,
        name="Diffuser",
    )

    circuit.h(range(number_of_qubits))
    circuit.x(range(number_of_qubits))

    target = number_of_qubits - 1
    controls = list(range(number_of_qubits - 1))

    if number_of_qubits == 1:
        circuit.z(0)
    else:
        circuit.h(target)
        circuit.append(
            MCXGate(len(controls)),
            controls + [target],
        )
        circuit.h(target)

    circuit.x(range(number_of_qubits))
    circuit.h(range(number_of_qubits))

    return circuit.to_gate()
```

Do not apply the diffuser to:

- clause qubits;
- XOR work qubits;
- the phase-output qubit;
- fixed classical clues.

## Running on IBM Quantum

Authenticate once:

```python
from qiskit_ibm_runtime import QiskitRuntimeService

QiskitRuntimeService.save_account(
    channel="ibm_quantum_platform",
    token="YOUR_IBM_QUANTUM_TOKEN",
    overwrite=True,
)
```

Load the service and select an accessible backend:

```python
from qiskit_ibm_runtime import QiskitRuntimeService

service = QiskitRuntimeService()
backend = service.backend("ibm_kingston")
```

Backend availability and account access can change. Replace `ibm_kingston` with another backend available to your account when necessary.

Transpile for the selected backend:

```python
from qiskit.transpiler.preset_passmanagers import (
    generate_preset_pass_manager,
)

pass_manager = generate_preset_pass_manager(
    target=backend.target,
    optimization_level=3,
)

isa_circuit = pass_manager.run(qc)

print("Logical circuit qubits:", qc.num_qubits)
print("Transpiled circuit qubits:", isa_circuit.num_qubits)
print("Transpiled depth:", isa_circuit.depth())
print("Transpiled operations:", isa_circuit.count_ops())
```

The total wall-clock time consists of:

- queue waiting time;
- compilation and transpilation;
- QPU initialization and execution;
- result processing.
### Unknown number of solutions

The optimal Grover iteration count depends on the number of valid solutions. If this number is unknown, do not choose a large iteration count merely to be safe. Use randomized iteration counts or estimate the solution count separately.

### Classical verification

Always verify the measured grid classically. A measured candidate must:

- preserve every given clue;
- contain digits `1, 2, 3, 4` in every row;
- contain digits `1, 2, 3, 4` in every column;
- contain digits `1, 2, 3, 4` in every 2×2 box.

## Expected result for the example

For

```text
1 0 3 4
3 4 1 2
2 1 4 3
4 3 2 1
```

the missing value is `2`, giving

```text
1 2 3 4
3 4 1 2
2 1 4 3
4 3 2 1
```
<img width="630" height="470" alt="fig1" src="https://github.com/user-attachments/assets/d95cc27a-deea-468f-a4f6-c560a8df4de3" />



