# SPIRE Quantum Readiness Analysis

## **Cryptographic Usage Locations**

### **1. Asymmetric Cryptography (NOT Quantum-Safe)**
- **RSA**: 2048/4096-bit keys
  - AWS KMS, GCP KMS, Azure Key Vault implementations
  - X.509 certificate generation
  - TPM device attestation

- **ECDSA**: P-256, P-384 curves
  - Primary signing algorithm across all cloud KMS providers
  - Server/agent certificate authorities
  - Node/workload attestation

- **Ed25519**:
  - `filippo.io/edwards25519` dependency

### **2. Hash Functions (NOT Quantum-Safe)**
- **SHA-256/384/512**: Primary hash algorithms
- **SHA-1**: Limited use (trust domain hashing in GCP KMS)

### **3. TLS (Partially Quantum-Safe)**
- **Standard**: TLS 1.2/1.3 with conventional ciphers
- **⚠️ EXPERIMENTAL QUANTUM-SAFE**: X25519Kyber768Draft00
  - Location: `pkg/common/tlspolicy/tlspolicy.go`
  - Configurable via `require_pq_kem` option
  - Hybrid post-quantum KEM (Key Encapsulation Mechanism)

### **4. JWT/JOSE Libraries (NOT Quantum-Safe)**
- `github.com/go-jose/go-jose/v4`
- `github.com/lestrrat-go/jwx/v3`
- `github.com/golang-jwt/jwt/v5`

### **5. Cloud KMS Dependencies**
- **AWS KMS**: `github.com/aws/aws-sdk-go-v2/service/kms`
- **GCP KMS**: `cloud.google.com/go/kms`
- **Azure Key Vault**: `github.com/Azure/azure-sdk-for-go/sdk/keyvault/azkeys`

### **6. Other Cryptographic Packages**
- **TPM**: `github.com/google/go-tpm`, `go-tpm-tools`
- **PKCS#7**: `github.com/fullsailor/pkcs7`, `github.com/smallstep/pkcs7`
- **ACME/AutoCert**: `golang.org/x/crypto/acme`
- **Core**: `golang.org/x/crypto` v0.48.0

## **Quantum-Safety Assessment**

**✅ QUANTUM-SAFE (1 component)**:
- **Kyber768** (X25519Kyber768Draft00) - Experimental hybrid PQ-KEM for TLS 1.3

**❌ NOT QUANTUM-SAFE (everything else)**:
- All signature algorithms (RSA, ECDSA, Ed25519)
- All hash functions (SHA family)
- All JWT/JWS operations
- Cloud KMS operations (limited by provider support)
- X.509 certificates
- TPM attestation

**Summary**: SPIRE has **experimental** post-quantum support for TLS key exchange only. All authentication, signing, and hashing operations use classical cryptography vulnerable to quantum attacks.
