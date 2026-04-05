# PQC Verification Benchmarks

## Methodology

**Iterations:** 1000 per algorithm per machine  
**Tool:** Native Rust binary using `near-crypto` crate (release build)  
**Each iteration:** Generate key pair → sign 64-byte message → verify signature  
**Metric:** Wall-clock nanoseconds per verification

Two hardware profiles tested — gas constants are calibrated to the **min-spec machine**.

---

## Hardware Profiles

### Machine A — Primary Validator
- CPU: Intel Core (Skylake), 4 vCPU
- RAM: 16GB
- Role: Block producer

### Machine B — Secondary Validator (Min-Spec)
- CPU: Intel Xeon (Skylake), 2 vCPU
- RAM: 4GB
- Role: Validation / attestation

---

## Results

### Machine A (4-core, 16GB)

| Algorithm | p50 | p95 | p99 | Max |
|---|---|---|---|---|
| FN-DSA (Falcon-512) | 0.041ms | 0.148ms | 0.192ms | 0.341ms |
| ML-DSA (Dilithium3) | 0.162ms | 0.521ms | 0.894ms | 1.203ms |
| SLH-DSA (SPHINCS+) | 0.626ms | 1.847ms | 2.203ms | 3.041ms |

### Machine B (2-core, 4GB) — Min-Spec

| Algorithm | p50 | p95 | p99 | Max |
|---|---|---|---|---|
| FN-DSA (Falcon-512) | 0.055ms | 0.187ms | 0.241ms | 0.398ms |
| ML-DSA (Dilithium3) | 0.330ms | 1.124ms | 1.703ms | 2.891ms |
| SLH-DSA (SPHINCS+) | 0.972ms | 3.614ms | 5.098ms | 7.203ms |

---

## Burst Test (FN-DSA)

5 FN-DSA transactions submitted in rapid succession, all accepted in the same block:

| TX | Gas used | Status |
|---|---|---|
| 1 | 4.0013 TGas | Executed |
| 2 | 4.0013 TGas | Executed |
| 3 | 4.0013 TGas | Executed |
| 4 | 4.0013 TGas | Executed |
| 5 | 4.0013 TGas | Executed |

All 5 succeeded. Block gas limit was not the bottleneck — **chunk size** is the limiting factor for PQC transactions due to large key/signature sizes.

---

## Throughput Analysis

### Chunk-Size Limited (not gas-limited)

An FN-DSA transaction is approximately **1,849 bytes** on-wire (666B signature + 897B public key + TX overhead).

With NEAR's default chunk size limit:
- **Chunk-size limit:** ~2,164 FN-DSA TXs per block
- **Gas limit equivalent:** ~6,428 FN-DSA TXs per block

The chunk-size limit is the binding constraint — FN-DSA blocks are chunk-limited, not gas-limited. This means:
- FN-DSA gas could be raised further without reducing real throughput
- SLH-DSA (~8KB signatures) is severely chunk-limited: ~250 TXs per block maximum

### Comparison Table

| Algorithm | TX size | TXs/block (chunk limit) | TXs/block (gas limit) | Binding limit |
|---|---|---|---|---|
| FN-DSA | ~1,849B | ~2,164 | ~6,428 | Chunk size |
| ML-DSA | ~5,300B | ~754 | ~3,000 | Chunk size |
| SLH-DSA | ~8,150B | ~490 | ~1,125 | Chunk size |

---

## Gas vs Real Compute

Using NEAR's 1 TGas ≈ 1ms model:

| Algorithm | Assigned gas | Time equivalent | Actual p99 (min-spec) | Safety factor |
|---|---|---|---|---|
| FN-DSA | 1.4 TGas | 1.4ms | 0.241ms | 5.8× |
| ML-DSA | 3.0 TGas | 3.0ms | 1.703ms | 1.76× |
| SLH-DSA | 8.0 TGas | 8.0ms | 5.098ms | 1.57× |

FN-DSA has the largest safety factor intentionally — it is the default algorithm and should remain inexpensive for ordinary users.
