# Flux PL/I — What Native BIT Strings Teach About Error Masks

PL/I (Programming Language One, 1964) was IBM's attempt to unite scientific (Fortran) and business (COBOL) computing in a single language. It runs on IBM mainframes and has a Linux compiler (Iron Spring). This repo implements the Flux constraint engine in PL/I.

## How to Read PL/I

PL/I reads like a more rigorous C with better type declarations — despite predating C by six years.

```pli
/* This is a comment */

DECLARE X FIXED BINARY(15);           /* 15-bit signed integer */
DECLARE Y DEC FIXED(15,4);            /* Exact decimal: 15 digits, 4 after point */
DECLARE MASK BIT(8);                  /* 8 bits — native bit string */
DECLARE NAME CHAR(32);                /* Fixed-length character string */

/* Structures — like C structs */
DECLARE 1 CONSTRAINT,
    2 LO    DEC FIXED(15,4),          /* Lower bound — exact decimal */
    2 HI    DEC FIXED(15,4),          /* Upper bound */
    2 SEV   FIXED BIN(8),             /* Severity */
    2 VIOLATED BIT(1);                /* Single bit: violated or not */

/* Arrays of structures */
DECLARE CONSTRAINTS(8) LIKE CONSTRAINT;

/* Bit operations — native, not simulated */
SUBSTR(MASK, 3, 1) = '1'B;           /* Set bit 3 — bit 3 is '1' Bit */

/* BOOL — arbitrary bitwise logic via truth table */
/* BOOL(A, B, '1111'B) = OR, '1000'B = AND, '0110'B = XOR */
MASK = BOOL(BLOCK1_MASK, BLOCK2_MASK, '1111'B);  /* OR */

/* Control flow */
DO I = 1 TO 8;
  IF SENSOR_VAL < CONSTRAINTS(I).LO THEN
    SUBSTR(MASK, I, 1) = '1'B;
END;

/* Procedures */
CHECK: PROCEDURE(VAL, IDX);
  DECLARE VAL DEC FIXED(15,4);
  DECLARE IDX FIXED BINARY(8);
  /* ... logic ... */
END CHECK;

/* Exception handling */
ON CONVERSION BEGIN;   /* Trap invalid data (PL/I's NaN analog) */
  /* handle bad conversion */
END;
```

Key ideas:
- **`BIT(n)` is a native type** — not an integer you shift, but a string of bits you address by position with `SUBSTR`.
- **`BOOL(A, B, truth_table)`** — arbitrary bitwise logic. The truth table IS the operation. `'1111'B` = OR, `'1000'B` = AND.
- **`DEC FIXED(p,q)`** — exact decimal arithmetic. No floating-point drift.
- **`ON CONDITION`** — exception handling. `ON CONVERSION` traps invalid data.
- **Structures** — COBOL-style records. Self-documenting data layout.

## How the Constraint Engine Maps to PL/I

| Constraint Engine Concept | PL/I Mechanism |
|---------------------------|----------------|
| Error mask (8 bits) | `BIT(8)` — native bit string, not simulated |
| Bitwise OR coalescence | `BOOL(M1, M2, '1111'B)` — truth-table OR |
| Constraint bounds | `DEC FIXED(15,4)` — exact decimal, zero drift |
| Single constraint violated | `BIT(1)` field in structure |
| Exception on bad data | `ON CONVERSION` handler |
| Pipeline stages | `PROCEDURE` blocks |

```
FLXCHECK.pli    — Core engine: bounds check, BIT(8) mask, severity
FLXFRACT.pli    — Fracture-coalesce: BFS blocks, BOOL OR coalescence
FLXSEDIMNT.pli  — Sediment layers: stack of bound corrections
FLXMAIN.pli     — Integration test: 30 assertions across 8 phases
```

## What PL/I Teaches Us

**The error mask should be a native type, not a simulation.** Every other language builds bitmasks from integer shift-and-OR. PL/I's `BIT(8)` IS the mask. There's no conceptual gap between "I need 8 bits" and the type declaration.

Compare:

```c
/* C — simulate the mask */
uint8_t mask = 0;
mask |= (1 << 2);           /* bit 3 = constraint 3 */
mask = block1 | block2;     /* OR coalescence */
```

```pli
/* PL/I — the mask IS the type */
DECLARE MASK BIT(8);
SUBSTR(MASK, 3, 1) = '1'B;                    /* bit 3 = constraint 3 */
MASK = BOOL(BLOCK1, BLOCK2, '1111'B);          /* OR coalescence */
```

The C version works. The PL/I version *declares intent*. `BIT(8)` doesn't just hold the data — it IS the data.

Three specific lessons:

1. **BOOL truth tables are the universal bitwise operation.** `BOOL(A, B, '1111'B)` is OR. `'1000'B` is AND. `'0110'B` is XOR. The 4-bit truth table encodes ANY binary Boolean function in one call. This is more general than any operator overloading.

2. **DEC FIXED eliminates IEEE 754 surprises.** When you say LO = -40.0 and HI = 85.0, that's exact — not "close enough." The constraint engine needs this. A bounds check that fails because of floating-point representation is a false violation.

3. **ON CONVERSION is constraint-safe error handling.** Invalid data doesn't crash or silently corrupt — it triggers a handler you define. This is the same pattern as the constraint engine's per-constraint violation tracking.

## Theorem: Coalescence Correctness

If fracture correctly identifies connected components of the constraint-dimension dependency graph, coalescence via bitwise OR preserves zero false negatives. Each constraint violation is a Boolean event. For independent blocks, event spaces are disjoint. Union = OR.

PL/I's `BOOL(M1, M2, '1111'B)` IS this theorem, expressed as a single builtin function call.

## Files

| File | Purpose |
|------|---------|
| `FLXCHECK.pli` | Core engine: bounds check, BIT(8) mask, severity |
| `FLXFRACT.pli` | Fracture-coalesce: BFS blocks, BOOL OR coalescence |
| `FLXSEDIMNT.pli` | Sediment layers: stack of bound corrections |
| `FLXMAIN.pli` | Integration test: 30 assertions across 8 phases |

## Build & Run

PL/I requires a compiler — IBM PL/I for MVS, Open PL/I (Linux), or Iron Spring PL/I. The syntax follows PL/I standard conventions (DECLARE, PROCEDURE, DO, IF-THEN-ELSE, SELECT, ON CONDITION, PUT SKIP LIST).

## Where to Go Next

- **flux-cobol** — COBOL uses arithmetic to simulate the bitmask. PL/I shows what happens when the mask is native.
- **flux-rpg** — RPG's indicators (*IN01–*IN08) are the same concept in a different form — boolean flags as error mask bits.
- **flux-snobol** — SNOBOL4 takes the opposite approach: no mask variable at all. The control flow IS the mask.
- **flux-docs** — Full documentation: error masks, fracture-coalesce, sediment, thermodynamic analogy.

## Author

Forgemaster ⚒️ — Constraint Theory Ecosystem, 2026-05-19
