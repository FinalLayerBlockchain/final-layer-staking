# Final Layer — Quantum-Resistant Blockchain

**Final Layer** is a production fork of [NEAR Protocol](https://near.org) that replaces the deprecated elliptic curve cryptography (Ed25519, secp256k1) with NIST-standardized post-quantum cryptographic signature schemes.

## Why Post-Quantum?

Elliptic curve cryptography (Ed25519, secp256k1) used in most blockchains today is vulnerable to Shor's algorithm running on a sufficiently powerful quantum computer. NIST finalized its first post-quantum standards in 2024 (FIPS 204, 205, 206). Final Layer implements all three.

## Supported Signature Schemes

| Algorithm | Standard | Family | Public Key | Signature | Security Level |
|---|---|---|---|---|---|
| **FN-DSA** (Falcon-512) | FIPS 206 | Lattice (NTRU) | 897 bytes | 666 bytes | Level 1 |
| **ML-DSA** (Dilithium3) | FIPS 204 | Lattice (Module) | 1952 bytes | 3309 bytes | Level 3 |
| **SLH-DSA** (SPHINCS+-128) | FIPS 205 | Hash-based | 32 bytes | ~8000 bytes | Level 1 |

Ed25519 and secp256k1 are removed from this fork.

## Architecture

Final Layer is built on NEAR Protocol's sharded architecture:
- **9 shards** for parallel transaction processing
- **~1 second** block times
- **NEAR VM** (WebAssembly) for smart contracts, extended with PQC host functions
- **Proof-of-Stake** consensus with quantum-resistant validator keys
- Native token: **FLC** (Final Layer Coin)

## Key Changes from NEAR Protocol

### 1. Cryptographic Layer (`near-crypto`)
- Replaced Ed25519/secp256k1 with FN-DSA, ML-DSA, SLH-DSA
- New `KeyType` variants: `MLDSA = 2`, `FNDSA = 3`, `SLHDSA = 4`
- Borsh serialization uses 4-byte LE length prefix for variable-length PQC keys
- Key format: `<algo>:<base58(bytes)>` (e.g., `fndsa:3abc...`)

### 2. VM Host Functions (`pqc_host_fns.rs`)
Three new host functions exposed to WASM contracts:
```
pqc_verify_fndsa(pk_ptr, pk_len, sig_ptr, sig_len, msg_ptr, msg_len) -> u64
pqc_verify_mldsa(pk_ptr, pk_len, sig_ptr, sig_len, msg_ptr, msg_len) -> u64
pqc_verify_slhdsa(pk_ptr, pk_len, sig_ptr, sig_len, msg_ptr, msg_len) -> u64
```

Gas costs (protocol v1003, calibrated on min-spec validator hardware):
- FN-DSA: **1.4 TGas** (p99 = 0.24ms on 2-core/4GB)
- ML-DSA: **3.0 TGas** (p99 = 1.70ms on 2-core/4GB)
- SLH-DSA: **8.0 TGas** (p99 = 5.10ms on 2-core/4GB)

### 3. Staking Pool Contract (`fl_staking_pool v5`)
Custom staking pool with:
- Time-based lockup (`LOCKUP_NS = 48h`)
- 4-epoch unbonding period
- PQC key registration via `parse_key_string()` with exact-length enforcement
- First-delegator phantom reward fix
- `claim_rewards()` resets lock timestamp

### 4. Protocol Versioning
- Protocol versions 1001–1003 (NEAR uses ~60–70)
- v1001: Genesis with PQC
- v1002: 9-shard deployment
- v1003: Gas rebalance (hard fork)

## Repository Structure

```
final-layer/
├── contracts/
│   └── staking-pool/
│       └── src/lib.rs          # PQC-aware staking pool contract
├── runtime/
│   ├── pqc_host_fns.rs         # PQC host function implementations + gas constants
│   └── version_notes.txt       # Protocol version history
├── docs/
│   ├── architecture.md         # System architecture
│   ├── pqc_algorithms.md       # PQC scheme comparison and rationale
│   ├── gas_rationale.md        # Why gas constants were set as they are
│   ├── staking.md              # Staking pool mechanics
│   ├── benchmarks.md           # PQC verification benchmarks
│   └── protocol_upgrades.md    # Hard fork procedure
└── scripts/
    └── deploy-validator.sh     # Example validator deployment
```

## Relation to nearcore

This repository contains the Final Layer-specific additions and contracts. The full node implementation is built on top of [`nearcore`](https://github.com/TechAdoptionGroup/nearcore) with the following crates modified:

- `core/crypto` — PQC key types
- `runtime/near-vm-runner/src/logic/` — PQC host functions
- `core/primitives-core/src/version.rs` — Protocol version

## License

Apache 2.0 (inherited from NEAR Protocol)
