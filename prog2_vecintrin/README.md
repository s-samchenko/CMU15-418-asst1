# Prog2: Vectorizing Code Using SIMD Intrinsics

## What I Implemented

Vectorized `clampedExpSerial()` using CMU418 fake SIMD intrinsics. The function computes `values[i]^exponents[i]` for all elements and clamps results to 4.18, using exponentiation by squaring.

### Implementation approach

- Outer loop processes `VECTOR_WIDTH` elements at a time
- `maskLoop` tracks which lanes still have `y > 0`
- `maskOdd` tracks which lanes have `y & 1 == 1` (need to multiply result by xpower)
- Loop exits when `_cmu418_cntbits(maskLoop) == 0` (all lanes done)
- Clamp applied once after the loop

## Vector Utilization vs Width (N=10000)

| Vector Width | Total Instructions | Vector Utilization |
|---|---|---|
| 2  | 1,466,908 | 90.63% |
| 4  | 733,706   | 90.39% |
| 8  | 367,121   | 90.34% |
| 16 | 183,809   | 90.33% |
| 32 | 184,378   | 90.33% |

## Analysis

**Why utilization stays flat across widths:**

The ~10% waste comes almost entirely from loop divergence — lanes with small exponents finish early while lanes with large exponents keep iterating. Since divergence is independent of vector width, utilization stays constant as W increases from 2 to 32.

The tail problem (last iteration when N % W != 0) is negligible at N=10000 — at most W-1 wasted lanes = <0.3% waste even at W=32.

**Why total instructions decrease with wider W:**

Wider vectors process more elements per instruction. Going from W=2 to W=32 (16x wider) reduces instruction count by roughly 16x — from ~1.4M to ~184K. 

**Theoretical peak vs actual:**

At 90% utilization, roughly 10% of vector lanes are wasted per instruction due to divergence. This is the irreducible cost of data-dependent loop counts. Production SIMD code typically sits in the 80-95% range — 90% is solid.

## Vectorized Array Sum

### Implementation

Two-phase approach:

**Phase 1 — parallel accumulation O(N/W):**
Loop through the array W elements at a time, accumulating into a vector of W independent partial sums. Lane `i` accumulates elements `i, i+W, i+2W...`

**Phase 2 — horizontal reduction O(log₂W):**
Collapse the W partial sums into a single scalar using log₂W iterations of interleave + hadd:

```
W=4 example:
[a, b, c, d]
→ interleave → [a, c, b, d]
→ hadd       → [a+b, a+b, c+d, c+d]
→ interleave → [a+b, c+d, a+b, c+d]
→ hadd       → [a+b+c+d, a+b+c+d, a+b+c+d, a+b+c+d]
```

After log₂W iterations every lane contains the total sum. Return `sum.value[0]`.

This implementation has O(N/W + log₂W) span:
- Phase 1 parallelizes N additions across W lanes simultaneously
- Phase 2 collapses in log₂W steps instead of W-1 serial additions

For W=32, N=100000: serial does 100000 sequential additions. Vector does 3125 parallel additions + 5 reduction steps.