# Quantum Readiness Analysis - Trustee Project

## **Cryptographic Packages & Locations**

### **Core Cryptographic Libraries**
1. **openssl** (0.10.75) - Used throughout for X.509, TLS, key operations
   - Locations: `kbs/`, `attestation-service/`, `deps/verifier/` (SNP, TPM, SE, Azure vTPM, Nvidia verifiers)

2. **aws-lc-rs** (via jsonwebtoken) - AWS LibCrypto for JWT operations
   - Locations: `kbs/src/token/jwk.rs`, `attestation-service/`, `deps/verifier/` (CCA, Nvidia)

3. **ring** - Low-level crypto primitives (transitive dependency)

### **Public Key Cryptography** ❌ NOT quantum-safe
4. **rsa** (0.9.10) - RSA encryption/signing with SHA-2
   - Locations: `kbs/src/jwe.rs` (RSA-OAEP-256, deprecated RSA1_5), `attestation-service/`

5. **p256** (0.13.2) - NIST P-256 elliptic curve + ECDH
   - Locations: `kbs/src/jwe.rs` (ECDH-ES+A256KW), `kbs/Cargo.toml`

6. **p521** (0.13.2) - NIST P-521 elliptic curve + ECDH
   - Locations: `kbs/Cargo.toml`

7. **elliptic-curve**, **ecdsa**, **sec1**, **pkcs8**, **spki** - EC infrastructure

### **Symmetric Cryptography** (Quantum-resistant for encryption, NOT for key exchange)
8. **aes-gcm** (0.10.1) - AES-256-GCM authenticated encryption
   - Locations: `kbs/src/jwe.rs`, `kbs_protocol` (guest-components), `crypto` crate

9. **aes-kw** (0.2.1) - AES Key Wrap
   - Locations: `kbs/src/jwe.rs`, `kbs_protocol`, `crypto` crate

10. **concat-kdf** (0.1.0) - Concatenation KDF for ECDH
    - Locations: `kbs/Cargo.toml`, `crypto` crate

11. **cbc**, **ctr**, **cipher** - Block cipher modes

### **Hash Functions** (Quantum-resistant)
12. **sha2** (0.10) - SHA-256/SHA-384/SHA-512
    - Locations: Workspace-wide, `rvps/` (in-toto), attestation verification

13. **digest** - Hash function traits

### **JWT/JWS/JWE** ❌ NOT quantum-safe
14. **jsonwebtoken** (10) - JWT with aws-lc-rs backend
   - Locations: `kbs/src/token/jwk.rs`, `attestation-service/`, `deps/verifier/` (CCA, Nvidia)

15. **josekit** (0.10.3) - JOSE framework (dev dependency)

16. **jwt-simple** - Via `kbs_protocol`

### **Hardware Attestation Crypto** ❌ NOT quantum-safe
17. **sev** (7.x) - AMD SEV-SNP certificate verification
    - Locations: `deps/verifier/src/snp/mod.rs`

18. **tss-esapi** (7.5) - TPM 2.0 cryptographic operations
    - Locations: `deps/verifier/src/tpm/`, Azure vTPM verifiers

19. **intel-tee-quote-verification-rs** - Intel SGX/TDX DCAP quote verification
    - Locations: `deps/verifier/src/intel_dcap/`

20. **az-snp-vtpm**, **az-tdx-vtpm**, **az-cvm-vtpm** - Azure vTPM crypto

21. **nvml-wrapper** - Nvidia GPU attestation

### **TLS/SSL** ❌ NOT quantum-safe
22. **rustls** (via reqwest/actix-web) - TLS 1.2/1.3 implementation

### **X.509/ASN.1**
23. **x509-parser** (0.18.1), **asn1-rs** (0.7.1) - Certificate parsing
    - Locations: SNP, Nvidia verifiers

24. **der**, **pem** - Encoding formats

### **Supporting Libraries**
25. **cryptoki** (0.12.0) - PKCS#11 interface (optional)
    - Locations: `kbs/src/plugins/implementations/pkcs11.rs`

26. **getrandom**, **rand** (0.8.5) - Secure random number generation

27. **subtle**, **zeroize**, **secrecy** - Constant-time operations & secure memory

## **Quantum-Safety Assessment**

### ❌ **NONE of the cryptography is quantum-safe:**

- **RSA**, **ECDSA (P-256/P-521)**, **ECDH** - Vulnerable to Shor's algorithm
- **All JWT/JWS signatures** - Use RSA or ECDSA
- **TLS handshakes** - Use ECDH/RSA key exchange
- **Hardware attestation** - SEV-SNP, TPM, SGX, TDX all use classical public-key crypto

### ✅ **Quantum-resistant components** (but insufficient alone):
- AES-256-GCM (symmetric encryption)
- SHA-2 family (hashing)

### **Post-quantum libraries**: **NONE found**
No usage of Kyber, Dilithium, SPHINCS+, Falcon, or any NIST PQC standards.
