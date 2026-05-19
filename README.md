# flux-pli — PL/I Constraint Engine

## What PL/I's BIT Strings Teach About Constraint Engines

PL/I (Programming Language One, 1964) was IBM's attempt to unite scientific
(Fortran) and business (COBOL) computing in a single language. What makes it
remarkable for constraint engines is **native BIT strings** — the error mask
isn't simulated, it's a first-class data type.

### The Key Insight

Every constraint engine needs an error mask: which constraints are violated.
In most languages, this is built from integer bitwise operations:

| Language | Error Mask Implementation |
|----------|--------------------------|
| C        | `uint8_t mask |= (1 << i)` — shift and OR |
| Python   | `mask |= (1 << i)` — same, slower |
| Rust     | `mask |= 1u8 << i` — same, safer |
| COBOL    | `ADD BIT-VAL TO RESULT-ERROR-MASK` — arithmetic simulation |
| **PL/I** | `SUBSTR(MASK, I, 1) = '1'B` — **native bit string** |

PL/I doesn't simulate bitmasks. `BIT(8)` IS a bitmask. `BOOL(A, B, '1111'B)`
IS bitwise OR. This was true in 1964 — before C existed.

### What PL/I Gets Right

1. **BIT(n) is a native type** — not an integer you shift, but a string of bits
   you address by position with `SUBSTR`. The error mask is declarative.

2. **BOOL builtin** — `BOOL(A, B, truth_table)` does arbitrary bitwise logic.
   `'1111'B` = OR, `'1000'B` = AND, `'0110'B` = XOR. The truth table IS the
   operation. No operator overloading needed.

3. **DEC FIXED(p,q)** — exact decimal arithmetic. No floating-point drift in
   constraint bounds. When you say LO = -40.0 and HI = 85.0, that's exact.

4. **ON CONVERSION** — exception mechanism that traps invalid data conversion
   (PL/I's NaN analog). Install a handler, catch bad data, continue.

5. **STRUCTURE** — COBOL-style records for constraint definitions. Group LO,
   HI, SEVERITY, VIOLATED in one structure — self-documenting data layout.

### Architecture

```
FLXCHECK.pli    — Core engine: bounds check, BIT(8) mask, severity
FLXFRACT.pli    — Fracture-coalesce: BFS blocks, BOOL OR coalescence
FLXSEDIMNT.pli  — Sediment layers: stack of bound corrections
FLXMAIN.pli     — Integration test: 30 assertions across 8 phases
```

### The Error Mask in PL/I

```pli
DECLARE ERROR_MASK BIT(8);           /* 8 bits, one per constraint */
DECLARE CONSTRAINTS(8) STRUCTURE(
    LO DEC FIXED(15,4),
    HI DEC FIXED(15,4),
    SEVERITY FIXED BIN(8),
    VIOLATED BIT(1));

/* Set bit 3 (constraint 3 violated) */
SUBSTR(ERROR_MASK, 3, 1) = '1'B;

/* Coalesce two block masks via OR */
ERROR_MASK = BOOL(BLOCK_MASK_1, BLOCK_MASK_2, '1111'B);
```

Compare with C:
```c
uint8_t error_mask = 0;
error_mask |= (1 << 2);  /* bit 3 = constraint 3 */
error_mask = block_mask_1 | block_mask_2;
```

The C version works. The PL/I version *declares intent*. The BIT string
doesn't just hold the data — it IS the data. There's no conceptual gap
between "I need 8 bits" and `BIT(8)`.

### BOOL Truth Tables

PL/I's `BOOL(A, B, truth_table)` uses a 4-bit truth table where each bit
represents the output for a combination of input bits:

| A | B | OR ('1111'B) | AND ('1000'B) | XOR ('0110'B) |
|---|---|-------------|--------------|--------------|
| 0 | 0 | 0           | 0            | 0            |
| 0 | 1 | 1           | 0            | 1            |
| 1 | 0 | 1           | 0            | 1            |
| 1 | 1 | 1           | 1            | 0            |

The truth table is literally the output column read as a binary number.
`'1111'B` = 15 = OR. This is elegant — any binary Boolean function in one call.

### Theorem: Coalescence Correctness

**Bitwise OR of independent block error masks = exact global error mask.**

Proof: Each constraint belongs to exactly one block (BFS partitioning).
Each violated constraint sets exactly one bit. OR preserves all set bits
from all operands. Therefore no violation is lost (zero false negatives)
and no violation is invented (zero false positives). QED.

PL/I's `BOOL(M1, M2, '1111'B)` IS this theorem, expressed as a single
builtin function call.

### Fracture-Coalesce Pattern

```
1. Build adjacency: ADJ(I,J) = '1'B if constraints share a dimension
2. BFS to find connected components (independent blocks)
3. Check each block independently
4. BOOL OR the block masks together → exact result
```

The BFS uses native BIT(1) for both the adjacency matrix and visited array.
PL/I was born to do this.

### Sediment Layers

Corrections to constraint bounds accumulate as geological layers:
- **PUSH_LAYER**: Add a correction on top of the stack
- **APPLY_SEDIMENT**: Apply all active layers bottom-to-top
- **POP_LAYER**: Remove top layer, revert bounds

The stack uses PL/I array indexing — no dynamic allocation needed for
the bounded case. Each layer records OLD and NEW bounds for audit trail.

### Running

PL/I requires a compiler. IBM PL/I for MVS, Open PL/I (Linux), or the
Iron Spring PL/I compiler can handle this code. The syntax follows
PL/I standard conventions:

- `DECLARE`, `PROCEDURE`, `DO`, `IF-THEN-ELSE`, `SELECT`
- `ON CONDITION` for exception handling
- `PUT SKIP LIST` for output
- `BOOL`, `SUBSTR` builtins for bit manipulation

### License

Part of the Flux Constraint Engine project — SuperInstance
