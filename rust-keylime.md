# Quantum Readiness Analysis: rust-keylime

## 1. Cryptographic Locations

### Dependencies (Cargo.toml)
- `openssl` (0.10.15) - primary crypto library
- `tss-esapi` (7.6.0) - TPM 2.0 operations
- `picky-asn1-der`, `picky-asn1-x509` - certificate handling
- `base64`, `hex`, `rand` - encoding/randomness

### Code Locations
| File | Cryptographic Operations |
|------|--------------------------|
| `keylime/src/crypto.rs` | RSA, ECC, AES-GCM, HMAC, PBKDF2, hashing |
| `keylime/src/crypto/x509.rs` | X.509 certificate generation |
| `keylime/src/crypto/symmkey.rs` | Symmetric key management |
| `keylime/src/tpm.rs` | TPM key generation, quotes, signatures |
| `keylime/src/hash_ek.rs` | EK public key hashing |
| `keylime-agent/src/keys_handler.rs` | Key encryption/decryption, HMAC |
| `keylime/src/ima/entry.rs` | IMA log hashing |
| `keylime/src/auth.rs` | Token hashing |
| `keylime/src/https_client.rs` | TLS client configuration |

## 2. Algorithms Used

### Asymmetric Cryptography
- RSA-1024/2048/3072/4096 (OAEP, PSS)
- ECC: P-192/224/256/384/521, SM2

### Symmetric Cryptography
- AES-128-GCM, AES-256-GCM

### Hashing
- SHA-1, SHA-256, SHA-384, SHA-512, SM3

### Key Derivation
- PBKDF2-HMAC-SHA1 (2000 iterations)

### Authentication
- HMAC-SHA384
- TLS/mTLS with OpenSSL

## 3. Quantum-Safety Assessment

### ❌ NOT Quantum-Safe

- **RSA** (all variants) - vulnerable to Shor's algorithm
- **ECC/ECDSA** (all curves including SM2) - vulnerable to Shor's algorithm
- **AES-128** - reduced security under Grover's algorithm
- **SHA-1/SHA-256** - weakened by Grover's algorithm
- **Current TLS/mTLS** - uses RSA/ECC key exchange
- **TPM signatures** (RSA-SSA, RSA-PSS, ECDSA) - vulnerable

### ⚠️ Partially Resistant

- **AES-256-GCM** - maintains ~128-bit security post-quantum (acceptable)
- **SHA-384/SHA-512** - larger output provides better resistance

### ✅ Post-Quantum Algorithms

- **NONE** - No NIST PQC algorithms (Kyber, Dilithium, SPHINCS+, etc.)

## Conclusion

**The codebase is NOT quantum-safe.** All asymmetric cryptography (RSA, ECC) and current TLS implementations are vulnerable to quantum attacks via Shor's algorithm. To achieve quantum-safety, the project would require migration to NIST-approved post-quantum algorithms like CRYSTALS-Kyber (key exchange) and CRYSTALS-Dilithium/SPHINCS+ (signatures).

## Migration Recommendations

To achieve quantum readiness:

1. **Replace RSA/ECC** with post-quantum algorithms:
   - CRYSTALS-Dilithium or SPHINCS+ for signatures
   - CRYSTALS-Kyber or FrodoKEM for key encapsulation

2. **Upgrade symmetric crypto**:
   - Use AES-256 exclusively (drop AES-128)
   - Consider SHA-384 or SHA-512 as minimum hash size

3. **Update TLS**:
   - Migrate to post-quantum TLS (hybrid mode initially)
   - Support PQ key exchange mechanisms

4. **TPM integration**:
   - Await TPM 2.0 spec updates for post-quantum algorithms
   - Consider hybrid classical/PQ schemes during transition

5. **Dependencies**:
   - Monitor OpenSSL 3.x for NIST PQC algorithm support
   - Consider libraries like `liboqs` (Open Quantum Safe) for PQ primitives
