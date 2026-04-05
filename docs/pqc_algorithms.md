# PQC Algorithm Reference

## FN-DSA (Falcon-512) — FIPS 206

Recommended for most users. Lattice-based using NTRU lattices, NIST Level 1 security (AES-128 equivalent).

Public key: 897 bytes. Signature: 666 bytes. Verification at p99 on a 2-core validator: 0.24ms. Gas: 1.4 TGas.

The smallest signatures of the three schemes. Fast verification. Compact on-chain footprint. FN-DSA is the default for wallets and delegators.

## ML-DSA (Dilithium3) — FIPS 204

Lattice-based using Module-LWE, NIST Level 3 security (AES-192 equivalent).

Public key: 1952 bytes. Signature: 3309 bytes. Verification at p99 on a 2-core validator: 1.70ms. Gas: 3.0 TGas.

Higher security level than FN-DSA at the cost of larger keys and signatures. Used by the secondary validators on the network.

## SLH-DSA (SPHINCS+-128) — FIPS 205

Hash-based using a Merkle hypertree structure, NIST Level 1.

Public key: 32 bytes. Signature: ~8000 bytes. Verification at p99 on a 2-core validator: 5.10ms. Gas: 8.0 TGas.

The most conservative security argument of the three — security depends only on hash function security, no lattice assumptions. Smallest public key. Largest signatures by a wide margin at ~8KB, which limits throughput due to chunk size constraints.

## Key encoding

All keys use `algo:base58(bytes)` format:

```
fndsa:34emUD68...
mldsa:2LtqmDcS...
slhdsa:2xGjfy...
```

In Borsh-serialized contexts, a 4-byte little-endian u32 length prefix is prepended before the raw key bytes. The staking contract validates exact byte lengths and panics on any mismatch — no silent truncation.

## Comparison

| | Ed25519 (removed) | FN-DSA | ML-DSA | SLH-DSA |
|---|---|---|---|---|
| Quantum-safe | No | Yes | Yes | Yes |
| Public key | 32B | 897B | 1952B | 32B |
| Signature | 64B | 666B | 3309B | ~8000B |
| Verify time (p99) | ~0.05ms | 0.24ms | 1.70ms | 5.10ms |
