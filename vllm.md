# Quantum Readiness Assessment - vLLM

## Cryptographic Usage Locations

### Direct Python Libraries (vLLM Code)

1. **hashlib (Python stdlib)** - Used in:
   - `vllm/utils/hashing.py`: SHA-256, MD5 (fallback)
   - `vllm/multimodal/hasher.py`: SHA-256, SHA-512
   - `vllm/entrypoints/openai/server_utils.py`: SHA-256 for token authentication

2. **secrets (Python stdlib)** - Used in:
   - `vllm/entrypoints/openai/server_utils.py`: constant-time comparison (`secrets.compare_digest`)

3. **ssl (Python stdlib)** - Used in:
   - `vllm/entrypoints/ssl.py`: SSL/TLS certificate management
   - `vllm/entrypoints/api_server.py`: SSL configuration
   - `vllm/entrypoints/openai/cli_args.py`: SSL settings

### Third-Party Cryptographic Dependencies

4. **blake3** - `requirements/common.txt:8`
   - `vllm/multimodal/hasher.py`: BLAKE3 hashing (configurable via `VLLM_MM_HASHER_ALGORITHM`)

5. **xxhash** (optional) - `requirements/test.txt:1330`
   - `vllm/utils/hashing.py`: xxHash128 for prefix caching

6. **cbor2** - `requirements/common.txt:47`
   - Used for serialization before hashing (not cryptographic itself)

### Indirect/Transitive Dependencies

7. **TLS/SSL via Python's ssl module** - Uses OpenSSL underneath
8. **gRPC** - Uses TLS for secure channels (referenced but used with `insecure=True` in tests)

---

## Quantum-Safety Assessment

### ❌ NOT Quantum-Safe

1. **SHA-256, SHA-512** - Vulnerable to Grover's algorithm (quadratic speedup). Still has acceptable security margin for now (128-bit effective strength for SHA-256).

2. **MD5** - Already broken classically, completely insecure.

3. **BLAKE3** - Similar to SHA-3, susceptible to Grover's algorithm but maintains good quantum resistance (256-bit → 128-bit effective).

4. **SSL/TLS (via OpenSSL)** - Uses:
   - RSA/ECDSA for key exchange/signatures - **BROKEN by Shor's algorithm**
   - AES for symmetric encryption - Reduced security (Grover's algorithm)
   - SHA-2 for MACs - Reduced security

5. **xxHash** - Non-cryptographic hash, not designed for security at all.

### ✅ Quantum Considerations

- **Hashing functions** (SHA-256/512, BLAKE3): Retain ~50% security strength post-quantum (still acceptable for non-signature uses)
- **Authentication tokens** (SHA-256 in `server_utils.py`): Acceptable if tokens are sufficiently long (256+ bits)
- **Symmetric encryption** (if any AES in TLS): Still secure with 256-bit keys
- **Constant-time comparison** (`secrets.compare_digest`): Quantum-safe, prevents timing attacks

---

## Summary

### Current State
- **All asymmetric cryptography (RSA, ECDSA in TLS)** is **NOT quantum-safe**
- **Hash functions** maintain partial security but are weakened
- **No post-quantum algorithms** (Kyber, Dilithium, SPHINCS+) are currently used
- **Primary risk**: SSL/TLS certificate authentication and key exchange protocols

### Risk Areas
1. **High Risk**: SSL/TLS connections for API servers and gRPC
2. **Medium Risk**: Hash-based authentication tokens (acceptable for now)
3. **Low Risk**: Content hashing for caching and deduplication (non-security critical)

### Recommendations
To achieve quantum readiness:
1. Replace SSL/TLS with post-quantum key exchange (e.g., Kyber) when available in Python's ssl module
2. Monitor NIST post-quantum cryptography standardization progress
3. Consider hybrid classical/post-quantum approaches for critical services
4. Increase hash output sizes where feasible (use SHA-512 over SHA-256)
5. No immediate action required for content hashing (non-cryptographic use case)
