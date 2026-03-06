# Cryptography Analysis for GKM Project

## 1. Direct Code Cryptography Usage

### api/v1alpha1/clustergkmcache_webhook.go:5-6, 220-242
- **crypto/hmac**: HMAC for mutation signature
- **crypto/sha256**: SHA-256 hashing
- **Quantum-safe**: ❌ **No** (vulnerable to Grover's algorithm - reduces 256-bit security to 128-bit)

### cmd/main.go:20,74 & agent/main.go:4,61
- **crypto/tls**: TLS configuration for webhooks/metrics servers
- **NextProtos**: HTTP/1.1 (HTTP/2 disabled by default)
- **Quantum-safe**: ❌ **No** (default Go TLS uses RSA/ECDSA)

---

## 2. Major Dependency Cryptography

### golang.org/x/crypto v0.46.0 (indirect)
**Location**: SSH, OpenPGP, TLS utilities

**Algorithms**:
- **Key Exchange**:
  - ✅ **ML-KEM768-X25519** (quantum-safe hybrid KEM - draft-kampanakis-curdle-ssh-pq-ke-05)
  - Curve25519, ECDH P-256/384/521, DH-14-SHA256, DH-16-SHA512 (non-quantum-safe)
- **Ciphers**: AES-128-GCM, AES-256-GCM, ChaCha20-Poly1305, AES-128/192/256-CTR
- **MACs**: HMAC-SHA256-ETM, HMAC-SHA512-ETM, HMAC-SHA256, HMAC-SHA512, HMAC-SHA1
- **Signatures**: RSA (PKCS1, PSS), ECDSA (P-256/384/521), Ed25519
- **Host Keys**: RSA-SHA256/512, ECDSA-256/384/521, Ed25519

**Quantum-safe**: ⚠️ **Partial** - ML-KEM768 available in SSH but NOT default; other algorithms not quantum-safe

### github.com/sigstore/cosign/v3 v3.0.3
**Location**: pkg/cosign/verify.go (image signature verification)

**Algorithms** (from sigstore/sigstore stack):
- **RSA**: PKCS1v15 (2048/3072/4096-bit), PSS (2048/3072/4096-bit)
- **ECDSA**: P-256-SHA256, P-384-SHA384, P-521-SHA512
- **Ed25519**: Signature algorithm
- **Hash**: SHA-256, SHA-384, SHA-512

**Quantum-safe**: ❌ **No**

### github.com/sigstore/rekor v1.4.3
**Purpose**: Transparency log verification for signed artifacts

**Algorithms**: RSA, ECDSA, Ed25519, SSH, PGP, Minisign, TUF signatures
**Quantum-safe**: ❌ **No**

### github.com/sigstore/sigstore-go v1.1.4
**Purpose**: Sigstore bundle verification (v3 format)

**Algorithms**: ECDSA, RSA, Ed25519, SCT verification, certificate validation
**Quantum-safe**: ❌ **No**

### github.com/sigstore/fulcio v1.8.3 (indirect)
**Purpose**: Code signing certificate authority

**Algorithms**: ECDSA (P-256), RSA (2048+), X.509 certificate extensions
**Quantum-safe**: ❌ **No**

### github.com/google/certificate-transparency-go v1.3.2 (indirect)
**Purpose**: Certificate transparency log verification

**Algorithms**: RSA, ECDSA for CT log signatures, Merkle tree hashing
**Quantum-safe**: ❌ **No**

### k8s.io/client-go v0.35.0
**Purpose**: Kubernetes API client with mutual TLS

**Algorithms**: Standard Go TLS (RSA, ECDSA, X.509), CSR generation, certificate rotation
**Quantum-safe**: ❌ **No**

### sigs.k8s.io/controller-runtime v0.21.0
**Purpose**: Kubernetes controller framework with webhooks

**Algorithms**: TLS 1.2/1.3 for webhooks, certificate watching, dynamic certificates
**Quantum-safe**: ❌ **No**

### google.golang.org/grpc v1.77.0
**Purpose**: gRPC communication (CSI driver)

**Algorithms**: Standard TLS 1.2/1.3 (RSA, ECDSA)
**Quantum-safe**: ❌ **No**

### github.com/containers/ocicrypt v1.2.1 (indirect)
**Purpose**: OCI image layer encryption/decryption

**Algorithms**: AES-CTR block cipher, PKCS7, PKCS11, PGP (RSA, ECDSA)
**Quantum-safe**: ❌ **No**

### github.com/containers/libtrust v0.0.0-20230121012942 (indirect)
**Purpose**: Docker content trust

**Algorithms**: RSA keys, ECDSA (P-256/384/521), JWS/JWK
**Quantum-safe**: ❌ **No**

### github.com/go-jose/go-jose/v4 v4.1.3 (indirect)
**Purpose**: JWT/JWE/JWS operations (OIDC authentication)

**Algorithms**: RSA (PKCS1, OAEP, PSS), ECDSA (P-256/384/521), AES-GCM, AES-CBC-HMAC
**Quantum-safe**: ❌ **No**

### github.com/theupdateframework/go-tuf v0.7.0 (indirect)
**Purpose**: The Update Framework for secure software updates

**Algorithms**: RSA, ECDSA, Ed25519 signatures
**Quantum-safe**: ❌ **No**

### github.com/digitorus/pkcs7 v0.0.0-20230818184609 (indirect)
**Purpose**: PKCS#7 signatures and encryption

**Algorithms**: RSA, DSA, ECDSA, AES, 3DES
**Quantum-safe**: ❌ **No**

---

## 3. Summary

**Total Cryptographic Packages**: 15+ (direct + transitive dependencies)

**Quantum-Safe Status**:
- ✅ **1 package** with quantum-safe support: `golang.org/x/crypto` (ML-KEM768-X25519 hybrid for SSH)
- ❌ **14+ packages** without quantum-safe algorithms
- ⚠️ **0 packages** using quantum-safe algorithms by default

**Critical Non-Quantum-Safe Usage Points**:
1. **Image Signature Verification** (Sigstore/Cosign) - RSA/ECDSA signatures
2. **TLS Communications** - RSA/ECDSA for all HTTPS (webhooks, metrics, K8s API)
3. **X.509 Certificates** - RSA/ECDSA for Kubernetes admission webhooks
4. **HMAC-SHA256** - Mutation signatures in webhook validation
5. **gRPC** - TLS 1.3 with RSA/ECDSA for CSI driver communication
6. **OIDC Authentication** - RSA/ECDSA JWT signatures (go-jose)
7. **Container Registry Auth** - TLS with RSA/ECDSA
8. **OCI Image Encryption** - RSA/ECDSA key wrapping (ocicrypt)

**Quantum Threat Assessment**:
- **Shor's Algorithm**: Breaks RSA, ECDSA, DH - affects 95%+ of project crypto
- **Grover's Algorithm**: Reduces SHA-256 security from 256-bit to 128-bit (still acceptable)
- **Harvest Now, Decrypt Later**: All TLS traffic vulnerable to future quantum attacks

---

## 4. Quantum-Readiness Recommendations

### Short-term (Current ecosystem constraints):
1. **Increase key sizes**: Use RSA-4096, ECDSA P-384+ where possible
2. **Monitor crypto-agility**: Ensure configuration allows algorithm updates
3. **Track NIST PQC standards**: Monitor adoption in Go, Kubernetes, Sigstore ecosystems

### Long-term (Post-quantum migration):
1. **TLS**: Wait for Go stdlib support of ML-KEM, ML-DSA (CRYSTALS-Dilithium), SPHINCS+
2. **Sigstore**: Requires ecosystem-wide migration to PQC signatures
3. **Kubernetes**: Requires upstream support for PQC in certificates, admission webhooks
4. **HMAC**: Increase to SHA-384 or SHA-512 (provides 192-bit/256-bit quantum security)

### Blockers to Quantum-Safety:
- No PQC support in Go stdlib TLS (as of Go 1.25)
- No PQC support in Kubernetes X.509 infrastructure
- No PQC support in Sigstore/Rekor/Fulcio ecosystem
- Limited PQC support in OCI/Docker ecosystem

**Overall Assessment**: The GKM project is **NOT quantum-safe** and cannot become quantum-safe until the entire Go/Kubernetes/Sigstore ecosystem adopts NIST PQC standards (ML-KEM, ML-DSA, SLH-DSA). This is likely a 3-5+ year timeline based on current standardization and implementation progress.
