# Final Layer

Final Layer is a fork of [NEAR Protocol](https://near.org) that swaps out Ed25519 and secp256k1 for NIST-standardized post-quantum cryptography. The goal is a production blockchain that stays secure against both classical and quantum adversaries.

The full node source lives in [nearcore-pq](https://github.com/FinalLayerBlockchain/nearcore-pq). This repo has the staking contract, documentation, and benchmarks.

## Supported signature schemes

Final Layer supports three NIST post-quantum standards. FN-DSA is the recommended default for most users — it has the smallest signatures and the most headroom in gas pricing. ML-DSA offers a higher security level at the cost of larger keys. SLH-DSA has the most conservative security argument (hash-based only, no lattice assumptions) but the largest signatures at ~8KB.

| Algorithm | Standard | Public key | Signature | Gas |
|---|---|---|---|---|
| FN-DSA (Falcon-512) | FIPS 206 | 897 bytes | 666 bytes | 1.4 TGas |
| ML-DSA (Dilithium3) | FIPS 204 | 1952 bytes | 3309 bytes | 3.0 TGas |
| SLH-DSA (SPHINCS+-128) | FIPS 205 | 32 bytes | ~8000 bytes | 8.0 TGas |

## Architecture

Final Layer inherits NEAR's sharded proof-of-stake design: 9 shards, ~1 second block times, WebAssembly smart contracts, Doomslug consensus. The main additions are three PQC host functions exposed to WASM contracts and a custom staking pool contract that handles PQC key registration, a 48-hour lockup period, and a 4-epoch unbonding window.

## Repository layout

```
contracts/staking-pool/src/lib.rs    Staking pool contract (v5)
runtime/pqc_host_fns.rs              PQC host function implementations
docs/architecture.md                 System architecture
docs/pqc_algorithms.md               Algorithm comparison and encoding details
docs/gas_rationale.md                How gas constants were derived
docs/staking.md                      Staking lifecycle and security properties
docs/benchmarks.md                   Benchmark results across two hardware profiles
docs/protocol_upgrades.md            Hard fork procedure
scripts/deploy-validator.sh          Validator setup reference
```

## Protocol

Chain ID `final-layer-mainnet`, protocol version 1003, native token FLC. Protocol v1003 was a hard fork that raised ML-DSA gas from 2.1 to 3.0 TGas and SLH-DSA from 3.2 to 8.0 TGas based on production benchmark data.

## License

Apache 2.0, inherited from NEAR Protocol.
