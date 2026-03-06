# Quantum Readiness Assessment - Guest Components

## **Cryptographic Locations & Packages**

### **Core Cryptographic Libraries**
1. **`rsa` v0.9.10** - RSA key operations with OAEP/PKCS1v15 padding
   - `attestation-agent/deps/crypto/src/rust/rsa.rs`
   - `attestation-agent/kbs_protocol/src/keypair.rs`
   - ❌ **NOT quantum-safe**

2. **`p256` v0.13.2** - ECDH/ECDSA on P-256 curve
   - `attestation-agent/deps/crypto/src/rust/ec.rs`
   - `confidential-data-hub/hub/src/secret/mod.rs`
   - ❌ **NOT quantum-safe**

3. **`openssl` v0.10.75** - General crypto provider
   - `attestation-agent/deps/crypto/src/native/` (RSA, EC, AES)
   - `ocicrypt-rs/src/blockcipher/`
   - ❌ **NOT quantum-safe** (for PKI operations)

4. **`ring` v0.17.14** - Crypto primitives & RNG
   - `confidential-data-hub/kms/src/plugins/aliyun/` (RSA-PKCS1-SHA256, HMAC-SHA1)
   - `image-rs`, `ocicrypt-rs` (random number generation)
   - ❌ **NOT quantum-safe** (for signature algorithms)

### **Symmetric Crypto**
5. **`aes-gcm` v0.10.3, `aes` v0.8.4, `ctr` v0.9.2** - AES-256-GCM/CTR
   - `attestation-agent/deps/crypto/src/rust/aes256gcm.rs`, `aes256ctr.rs`
   - `ocicrypt-rs/src/blockcipher/aes_ctr.rs`
   - `attestation-agent/coco_keyprovider/src/enc_mods/crypto.rs`
   - ⚠️ **Partially quantum-resistant** (symmetric encryption safe, but key exchange mechanisms use quantum-vulnerable algorithms)

6. **`aes-kw` v0.2.1** - AES key wrapping
   - `attestation-agent/deps/crypto/src/rust/ec.rs` (ECDH-ES-A256KW)
   - ⚠️ **Partially quantum-resistant** (depends on key derivation)

### **Hash Functions**
7. **`sha2` v0.10.9** - SHA-256/384/512
   - Used throughout for digests, HMAC, KDFs
   - ✅ **Quantum-resistant** (Grover's algorithm only provides quadratic speedup)

8. **`hmac` v0.12.1** - HMAC construction
   - `ocicrypt-rs/src/blockcipher/aes_ctr.rs`
   - `confidential-data-hub/kms/` (HMAC-SHA1 legacy)
   - ✅ **Quantum-resistant** (when used with quantum-resistant hashes)

### **Digital Signatures & Authentication**
9. **`jwt-simple` v0.12.14** - JWT with Ed25519
   - `attestation-agent/coco_keyprovider/` (Ed25519 keypairs)
   - `attestation-agent/kbs_protocol/src/token_provider/`
   - ❌ **NOT quantum-safe** (EdDSA vulnerable to Shor's algorithm)

10. **`sequoia-openpgp` v2.2.0** - OpenPGP implementation
    - `image-rs/src/signature/policy/simple/verify.rs`
    - ❌ **NOT quantum-safe** (RSA/ECC signatures)

11. **`sigstore` v0.13.0** - Sigstore/Cosign signatures
    - `image-rs/Cargo.toml` (cosign image verification)
    - ❌ **NOT quantum-safe** (ECDSA/RSA signatures)

12. **`josekit` v0.10.3** - JOSE (JWE/JWS)
    - `ocicrypt-rs/src/keywrap/jwe.rs` (RSA-OAEP, ECDH-ES-A256KW)
    - ❌ **NOT quantum-safe** (RSA/ECDH for key agreement)

### **TLS/Transport**
13. **rustls, native-tls** - TLS implementations
    - Used in `reqwest`, `oci-client`, `sigstore`
    - ❌ **NOT quantum-safe** (conventional PKI/ECDHE)

### **Key Derivation**
14. **`concat-kdf` v0.1.0** - Concat KDF
    - `attestation-agent/deps/crypto/src/rust/ec.rs`
    - ⚠️ **Partially quantum-resistant** (ECDH inputs are vulnerable)

## **Quantum Safety Summary**
❌ **NO quantum-safe cryptography is used in this codebase.**

**Vulnerable components:**
- All public-key operations (RSA, ECDSA, ECDH, Ed25519)
- All digital signatures (JWT, PGP, Cosign)
- All key exchange mechanisms
- TLS connections

**Migration needed for post-quantum security:** Kyber (ML-KEM), Dilithium (ML-DSA), SPHINCS+, or hybrid approaches.
