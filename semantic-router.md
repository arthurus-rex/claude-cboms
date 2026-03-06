# Quantum Readiness Assessment: Semantic Router

**Assessment Date:** 2026-03-05

## Executive Summary

This codebase uses cryptography primarily for **transport security** (TLS/HTTPS) and **non-security hashing** (cache keys). **None of the cryptographic functions are currently quantum-safe.** All public-key cryptography (RSA, ECDSA, X25519) used in TLS connections is vulnerable to quantum attacks via Shor's algorithm.

---

## Cryptography Usage Locations

### Go (Standard Library)

#### TLS/X.509 Certificate Generation
- **Location:** `src/semantic-router/pkg/utils/tls/tls.go:36`
- **Algorithm:** RSA 4096-bit
- **Usage:** Self-signed certificate generation
- **Quantum-Safe:** ❌ NO - Vulnerable to Shor's algorithm

#### TLS Clients
- `src/semantic-router/pkg/extproc/server.go` - gRPC with TLS
- `src/semantic-router/pkg/responsestore/redis_store.go` - Redis TLS connections
- `src/semantic-router/pkg/routerreplay/store/redis.go` - Redis TLS connections
- **Quantum-Safe:** ❌ NO - Default TLS 1.2/1.3 uses RSA/ECDSA/X25519

#### Hashing (Non-Cryptographic Use)
- **MD5:**
  - `src/semantic-router/pkg/cache/redis_cache.go:5`
  - `src/semantic-router/pkg/cache/milvus_cache.go:5`
  - **Usage:** Cache key generation only
  - **Quantum-Safe:** ⚠️ PARTIALLY - Cryptographically broken but quantum-resistant for hashing; not used for security

- **SHA-256:**
  - `src/semantic-router/pkg/extproc/req_filter_rag_cache.go:4`
  - `src/semantic-router/pkg/modelselection/trainer.go:20`
  - **Usage:** Cache key generation and content addressing
  - **Quantum-Safe:** ⚠️ PARTIALLY - 128-bit quantum collision security (Grover's algorithm reduces from 256-bit to 128-bit)

#### Random Number Generation
- `src/semantic-router/pkg/responseapi/id.go:4` - crypto/rand
- `src/semantic-router/pkg/routerreplay/store/memory.go:5` - crypto/rand
- `dashboard/backend/handlers/openclaw_helpers.go:4` - crypto/rand
- **Quantum-Safe:** ✅ YES - CSPRNG, quantum-safe for random generation

### Rust Dependencies

#### TLS Libraries (via tokenizers, hf-hub for HTTPS)

**ring 0.17.14**
- **Locations:** `candle-binding/Cargo.lock:1568`, `onnx-binding/Cargo.lock:1568`
- **Algorithms:** RSA, ECDSA, Ed25519, X25519, AES-GCM, SHA-2, etc.
- **Usage:** TLS primitives for HTTPS model downloads
- **Quantum-Safe:** ❌ NO - Public-key algorithms vulnerable to Shor's algorithm

**rustls 0.23.36**
- **Locations:** `candle-binding/Cargo.lock:2746`, `onnx-binding/Cargo.lock:1639`
- **Algorithms:** TLS 1.2/1.3 with RSA/ECDSA/X25519
- **Usage:** TLS library for model downloads from Hugging Face
- **Quantum-Safe:** ❌ NO - Default ciphersuites are quantum-vulnerable

**openssl 0.10.75**
- **Location:** `onnx-binding/Cargo.lock:1214`
- **Algorithms:** Full OpenSSL suite (RSA, ECDSA, etc.)
- **Usage:** TLS/crypto bindings
- **Quantum-Safe:** ❌ NO - Classic public-key cryptography

**native-tls**
- **Location:** `nlp-binding/Cargo.lock`
- **Algorithms:** Platform TLS (delegates to OS)
- **Usage:** TLS abstraction layer
- **Quantum-Safe:** ❌ NO - Depends on OS TLS implementation

#### Hashing Libraries
- **ahash, fxhash, foldhash** - Non-cryptographic hash functions for data structures
- **crypto-common** - Cryptographic primitive traits
- **hashbrown** - HashMap implementation (non-cryptographic)

### Indirect Dependencies

**golang.org/x/crypto**
- Pulled in by gRPC, networking libraries
- Not directly imported in application code
- **Quantum-Safe:** ❌ NO - Standard Go crypto primitives

---

## Quantum-Safety Assessment by Category

### ❌ NOT Quantum-Safe

| Component | Algorithm | Threat |
|-----------|-----------|--------|
| RSA 4096-bit (self-signed certs) | Public-key encryption | Shor's algorithm breaks in polynomial time |
| TLS 1.2/1.3 (Go standard lib) | RSA/ECDSA/X25519 key exchange | Quantum computer can break handshake |
| Ring/Rustls (Rust) | RSA/ECDSA/Ed25519/X25519 | All EC and RSA schemes vulnerable |
| OpenSSL bindings | Classic public-key crypto | Full suite vulnerable |

### ⚠️ Partially Safe / Context-Dependent

| Component | Notes |
|-----------|-------|
| MD5 | Broken classically (collision attacks), but only used for cache keys (no security impact). Quantum-resistant for one-way hashing. |
| SHA-256 | 256-bit classical security, 128-bit quantum security (Grover's algorithm). Adequate for cache keys, marginal for long-term security. |

### ✅ Quantum-Safe

| Component | Notes |
|-----------|-------|
| crypto/rand (CSPRNG) | Random number generation remains secure |
| AES-256 (symmetric) | Would have 128-bit quantum security via Grover's algorithm (not currently used for data-at-rest) |

---

## Risk Analysis

### Current Threat Surface

1. **TLS Connections:**
   - Model downloads from Hugging Face (HTTPS via rustls/ring)
   - Redis connections (Go TLS)
   - gRPC external processor communication (Go TLS)
   - Dashboard WebSocket connections (Go TLS)

2. **Attack Scenario (Harvest Now, Decrypt Later):**
   - Adversary records encrypted TLS traffic today
   - When large-scale quantum computers exist, decrypts recorded sessions
   - **Impact:** Exposure of model files, cache data, API requests

3. **Non-Security Hashing:**
   - MD5/SHA-256 used only for cache keys
   - No authentication, integrity checking, or security-critical use
   - **Impact:** Minimal - cache collisions don't compromise security

### Low-Risk Areas

- No digital signatures for authentication
- No public-key encryption of application data
- No certificate pinning or PKI validation beyond standard TLS
- Hashing used only for performance (caching), not security

---

## Quantum-Safe Migration Path

### Short-Term (Pre-Quantum Era)

1. **Monitor NIST PQC Standardization:**
   - ML-KEM (Kyber) - Key encapsulation
   - ML-DSA (Dilithium) - Digital signatures
   - SLH-DSA (SPHINCS+) - Stateless signatures

2. **Dependency Updates:**
   - Track Go 1.x for post-quantum TLS support (in development)
   - Monitor rustls PQC integration (experimental as of 2026)
   - Follow OpenSSL 3.x post-quantum roadmap

### Medium-Term (Quantum-Safe Transition)

1. **TLS 1.3 + Post-Quantum:**
   - Migrate to hybrid key exchange (X25519 + ML-KEM)
   - Update Go TLS to use post-quantum ciphersuites
   - Update Rust dependencies (rustls + ring) to PQC-enabled versions

2. **Certificate Infrastructure:**
   - Replace RSA 4096 with ML-DSA (Dilithium) for self-signed certs
   - Support hybrid certificates (RSA + ML-DSA) during transition

3. **Hash Function Upgrades:**
   - Replace MD5 with SHA3-256 or BLAKE3
   - Consider SHA-256 → SHA3-512 for future-proofing (512-bit quantum security)

### Long-Term (Post-Quantum Era)

1. **Full Post-Quantum Migration:**
   - Remove all classical public-key crypto (RSA, ECDSA, X25519)
   - Use pure ML-KEM + ML-DSA for TLS
   - Validate against NIST PQC standards

2. **Code Locations to Update:**
   ```
   src/semantic-router/pkg/utils/tls/tls.go → Use ML-DSA key generation
   candle-binding/Cargo.toml → Update hf-hub with PQC-enabled rustls
   onnx-binding/Cargo.toml → Update tokenizers with PQC TLS
   All go.mod files → Update to Go with native PQC TLS support
   ```

---

## Recommendations

### Priority 1 (Immediate)
- ✅ Current cryptography is adequate for 2026 threat landscape
- Monitor NIST PQC library availability in Go/Rust ecosystems

### Priority 2 (1-2 years)
- Test hybrid TLS implementations (classical + PQC) in staging
- Evaluate performance impact of PQC algorithms (larger keys/signatures)

### Priority 3 (3-5 years)
- Implement full post-quantum TLS once standardized in Go/Rust
- Replace self-signed RSA certs with ML-DSA

### Priority 4 (Optional)
- Replace MD5 cache hashing with SHA3-256 (no security benefit, but future-proof)

---

## Conclusion

**The semantic-router codebase is NOT quantum-ready.** All public-key cryptography relies on quantum-vulnerable algorithms (RSA, ECDSA, X25519). However, cryptography is limited to transport security (TLS), with no application-layer encryption or signing.

**Quantum risk is LOW in the short term** (no high-value secrets encrypted with public-key crypto), but increases with "harvest now, decrypt later" attacks on recorded TLS traffic.

**Migration path is straightforward:** Wait for post-quantum TLS support in Go standard library and Rust crates (rustls/ring), then update dependencies. No application code changes required beyond certificate generation logic.
