# Quantum-Readiness Analysis of Keylime

**Analysis Date:** March 4, 2026
**Keylime Version:** 7.14.1
**Analyst:** Comprehensive Cryptographic Audit

---

## Executive Summary

Keylime currently relies heavily on **classical public-key cryptography (RSA, ECDSA)** that is **not quantum-safe**. While symmetric cryptography (AES-256, HMAC-SHA384) and hash functions (SHA-256/384) provide acceptable quantum resistance, the asymmetric algorithms used for digital signatures, key encapsulation, and TPM operations are all vulnerable to quantum attacks via Shor's algorithm.

**Key Findings:**
- ❌ RSA (1024-4096 bits) - vulnerable to quantum attacks
- ❌ ECDSA/ECC (P-256, P-384, P-521) - vulnerable to quantum attacks
- ❌ TLS key exchange (ECDHE/DHE) - vulnerable to quantum attacks
- ✅ AES-256-GCM - quantum-resistant (128-bit security)
- ✅ SHA-256/384/512 - quantum-resistant for most uses
- ✅ HMAC-SHA384 - quantum-resistant

**Recommended Action:** Begin post-quantum cryptography (PQC) migration immediately, targeting hybrid PQC deployment by Q4 2026.

---

## Table of Contents

1. [All Cryptographic Locations](#1-all-cryptographic-locations)
2. [Quantum-Safety Assessment](#2-quantum-safety-assessment)
3. [Quantum-Readiness Recommendations](#3-quantum-readiness-recommendations)
4. [Implementation Timeline](#implementation-timeline)
5. [Risk Assessment](#risk-assessment)
6. [Estimated Effort](#estimated-effort)
7. [Conclusion](#conclusion)

---

## 1. All Cryptographic Locations

### Core Cryptographic Dependencies

- **cryptography** (Python library) - primary crypto library >=3.3.2
- **hashlib** - hash functions (SHA-1, SHA-256, SHA-384, SHA-512)
- **hmac** - message authentication codes
- **ssl** - TLS/SSL implementation
- **secrets/os.urandom** - cryptographically secure random number generation
- **python-gnupg** - GPG signature verification

### Key Cryptographic Modules

| File Path | Primary Cryptographic Operations |
|-----------|----------------------------------|
| `keylime/crypto.py` | RSA encrypt/decrypt, RSA sign/verify, AES-GCM, PBKDF2, HMAC, token hashing |
| `keylime/ca_impl_openssl.py` | X.509 cert generation, RSA/ECC key generation, certificate signing (SHA-256) |
| `keylime/ca_util.py` | Certificate validation, private key encryption, CRL management |
| `keylime/cert_utils.py` | X.509 parsing, ECDSA/RSA signature verification, TPM cert validation |
| `keylime/common/algorithms.py` | Algorithm definitions (RSA, ECC, SHA families) |
| `keylime/tpm/tpm2_objects.py` | TPM algorithm constants, ECC curve definitions |
| `keylime/tpm/tpm_util.py` | TPM signature verification, credential encryption, ECDSA/RSA-PSS |
| `keylime/ima/file_signatures.py` | IMA signature verification (RSA, ECDSA) |
| `keylime/signing.py` | ECDSA verification, PGP verification, DSSE |
| `keylime/dsse/ecdsa.py` | ECDSA signing/verification (SECP256R1) |
| `keylime/dsse/x509.py` | X.509 ECDSA certificates |
| `keylime/tee/snp.py` | SEV-SNP ECDSA verification (P-384, SHA-384) |
| `keylime/web_util.py` | TLS context creation (TLS 1.2+) |
| `keylime/tornado_requests.py` | HTTPS/TLS client connections |

### Asymmetric Cryptography Usage

**RSA:**
- Key sizes: 1024, 2048 (default), 3072, 4096 bits
- Padding: OAEP (encryption), PKCS1v15 (signatures), PSS (signatures)
- Hash: SHA-1 (legacy), SHA-256
- Usage: Key generation, digital signatures, encryption, X.509 certificates

**Elliptic Curve Cryptography (ECC/ECDSA):**
- Curves: P-192, P-224, P-256 (default), P-384, P-521, SECP256R1, SECP384R1
- Usage: ECDSA signatures, ECC key generation, TPM attestation, DSSE signing, SEV-SNP

### Symmetric Cryptography Usage

**AES-256-GCM:**
- Key size: 256 bits
- Mode: GCM (Galois/Counter Mode) - authenticated encryption
- Block size: 16 bytes
- IV: Random generation for each encryption
- Usage: Payload encryption, credential blob encryption, private key encryption

### Hash Functions

- **SHA-1**: Legacy support in IMA, RSA OAEP padding (⚠️ deprecated)
- **SHA-256**: Primary hash - certificate signing, TPM operations, HMAC, fingerprints
- **SHA-384**: HMAC operations, SEV-SNP attestation
- **SHA-512**: Available for high-security scenarios
- **SM3-256**: Chinese standard (limited usage)
- **MD5**: Legacy support only (⚠️ deprecated)

### Key Derivation Functions

**PBKDF2-HMAC-SHA256:**
- Iterations: 600,000 (token hashing), 100,000 (legacy)
- Salt: 16 bytes (128-bit) per NIST recommendations
- Usage: Token hashing, secure storage, private key encryption

**Other KDFs:**
- **ConcatKDFHash**: TPM-related key derivation
- **KBKDFHMAC**: Counter mode KDF with HMAC

### Digital Signature Schemes

- **RSASSA**: RSA Signature Scheme with Appendix (PKCS1v15 padding, SHA-256)
- **RSAPSS**: RSA Probabilistic Signature Scheme (PSS padding, MGF1-SHA-256)
- **ECDSA**: Elliptic Curve Digital Signature Algorithm (SHA-256)
- **ECDAA**: Elliptic Curve Direct Anonymous Attestation
- **EC-Schnorr**: Schnorr signatures on elliptic curves

### TLS/Transport Security

- **TLS Version**: TLS 1.2 minimum (configurable at `web_util.py:216`)
- **TLS Cipher Suites**: Uses Python's `ssl.create_default_context()` (strong ciphers)
- **Certificate Verification**: CERT_REQUIRED with mutual TLS (mTLS)
- **Key Exchange**: ECDHE or DHE (cipher suite dependent)
- **Hostname Verification**: Disabled (authentication via client certificates)

### Random Number Generation

- `secrets.token_bytes()` - cryptographically secure random bytes
- `os.urandom()` - system entropy source
- `secrets.token_urlsafe()` - URL-safe random tokens

### TPM-Related Cryptographic Operations

**AIK Encryption with EK:**
- TPM2_MakeCredential operation
- Challenge generation with random 32-byte value
- AES encryption of credentials
- Base64 encoding of encrypted credential blob

**TPM Quote Verification:**
- Hash algorithms: SHA-1, SHA-256, SHA-384, SHA-512
- Signature algorithms: RSASSA, RSAPSS, ECDSA
- Signature verification with PSS or PKCS1v15 padding
- Attestation data integrity checking

**TPM Key Attributes:**
- Object attributes: FIXEDTPM, SENSITIVEDATAORIGIN, SIGN_ENCRYPT
- Key usage restrictions
- Restricted object handling

---

## 2. Quantum-Safety Assessment

### ❌ VULNERABLE TO QUANTUM ATTACKS (Shor's Algorithm)

The following algorithms used in Keylime are **NOT quantum-safe** and will be broken by sufficiently powerful quantum computers:

#### Asymmetric Cryptography

**1. RSA (All Key Sizes)**

- **Key Sizes**: 1024, 2048, 3072, 4096 bits
- **Locations**:
  - Core encryption/decryption (`crypto.py`)
  - Certificate signing (`ca_impl_openssl.py`)
  - TPM operations (`tpm_util.py`)
  - IMA signatures (`file_signatures.py`)
- **Usage**: Key generation, digital signatures (RSASSA, RSAPSS), encryption (OAEP), X.509 certificates
- **Vulnerability**: Shor's algorithm can factor large numbers in polynomial time
- **Impact**: **HIGH** - Core to PKI infrastructure, TPM attestation, encrypted payloads
- **Break Timeline**: Large-scale quantum computers (2030-2040 estimates)

**2. Elliptic Curve Cryptography (ECC/ECDSA)**

- **Curves Used**: P-192, P-224, P-256 (default), P-384, P-521, SECP256R1, SECP384R1
- **Locations**:
  - Digital signatures (`cert_utils.py`, `dsse/ecdsa.py`, `tpm_util.py`)
  - SEV-SNP attestation (`tee/snp.py`)
  - Certificates (`dsse/x509.py`)
- **Usage**: ECDSA signatures, ECC key generation, TPM attestation, DSSE signing
- **Vulnerability**: Shor's algorithm solves discrete logarithm problem on elliptic curves
- **Impact**: **HIGH** - Used for attestation signatures, certificate signing, DSSE envelope verification
- **Break Timeline**: Potentially faster to break than RSA with quantum computers

**3. TLS Key Exchange**

- **Protocols**: TLS 1.2+ uses ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) or DHE
- **Locations**: `web_util.py`, `tornado_requests.py`, all HTTPS communications
- **Vulnerability**: ECDHE/DHE key exchanges vulnerable to quantum attacks
- **Impact**: **HIGH** - All verifier-registrar-agent communications use TLS
- **Threat**: "Harvest now, decrypt later" - adversaries can record encrypted traffic and decrypt later

#### Digital Signature Schemes (Non-Quantum-Safe)

All current signature schemes are vulnerable:
- RSASSA (RSA Signature Scheme with Appendix)
- RSAPSS (RSA Probabilistic Signature Scheme)
- ECDSA (Elliptic Curve Digital Signature Algorithm)
- ECDAA (Elliptic Curve Direct Anonymous Attestation)
- EC-Schnorr

### ✅ QUANTUM-RESISTANT / ACCEPTABLE

The following cryptographic primitives are considered quantum-resistant or provide acceptable quantum security:

#### Symmetric Cryptography

**1. AES-256-GCM**

- **Location**: `crypto.py` (encrypt/decrypt functions)
- **Quantum Security Level**: ~128-bit (Grover's algorithm provides quadratic speedup)
- **Status**: ✅ **ACCEPTABLE** - 128-bit quantum security is generally considered sufficient
- **Usage**: Payload encryption, credential blob encryption, private key encryption
- **Note**: AES-256 provides 128-bit quantum security (half the classical 256-bit security)

#### Hash Functions

**1. SHA-256**

- **Locations**: Throughout codebase - certificate signing, TPM operations, HMAC, fingerprints
- **Quantum Security Level**: ~128-bit collision resistance (Grover's algorithm)
- **Status**: ✅ **ACCEPTABLE** for most uses
- **Caveat**: ⚠️ For long-term integrity (>20 years), consider SHA-384 or SHA-512

**2. SHA-384**

- **Locations**: HMAC operations (`crypto.py`), SEV-SNP attestation (`tee/snp.py`)
- **Quantum Security Level**: ~192-bit collision resistance
- **Status**: ✅ **GOOD** - Strong quantum resistance

**3. SHA-512**

- **Quantum Security Level**: ~256-bit collision resistance
- **Status**: ✅ **EXCELLENT** quantum resistance

**4. SHA-1** (⚠️ **DEPRECATED**)

- **Locations**: Legacy support in IMA (`file_signatures.py`), RSA OAEP padding (`crypto.py`)
- **Status**: ❌ **WEAK** even against classical attacks
- **Recommendation**: Remove immediately

#### Key Derivation

**1. PBKDF2-HMAC-SHA256**

- **Iterations**: 600,000 (token hashing), 100,000 (legacy)
- **Status**: ✅ **ACCEPTABLE**
- **Note**: Iteration count provides classical brute-force resistance; underlying HMAC-SHA256 is quantum-resistant

#### Message Authentication

**1. HMAC-SHA384**

- **Location**: `crypto.py` (do_hmac function)
- **Status**: ✅ **GOOD** - Quantum-resistant with 192-bit security

#### Random Number Generation

**1. secrets module / os.urandom**

- **Status**: ✅ **ACCEPTABLE**
- **Note**: Quality depends on OS entropy source, not directly quantum-vulnerable
- **Future Enhancement**: Consider quantum random number generators (QRNG) for enhanced entropy

---

## 3. Quantum-Readiness Recommendations

### CRITICAL PRIORITY (P0) - Foundation for Quantum Migration

#### 1. Develop Post-Quantum Cryptography (PQC) Roadmap

- **Action**: Create a formal PQC migration plan with phases, milestones, and timelines
- **Timeline**: Immediate (Q2 2026)
- **Stakeholders**: Security team, architecture team, TPM/attestation experts, community
- **Deliverables**:
  - PQC migration strategy document
  - Risk assessment
  - Resource allocation plan
  - Community communication plan

#### 2. Adopt NIST-Standardized PQC Algorithms

The following algorithms were standardized by NIST in 2024 and should be integrated:

**For Digital Signatures (Replace RSA/ECDSA):**

**ML-DSA (Module-Lattice-Based Digital Signature Algorithm, formerly Dilithium)**

- **Standard**: FIPS 204
- **Use Cases**: Certificate signing, attestation signatures, DSSE signatures
- **Parameter Sets**:
  - ML-DSA-44: ~2420 bytes public key, ~2420 bytes signature (128-bit security)
  - ML-DSA-65: ~4032 bytes public key, ~3309 bytes signature (192-bit security) ✅ **Recommended**
  - ML-DSA-87: ~4896 bytes public key, ~4627 bytes signature (256-bit security)
- **Impact**: Replace ECDSA/RSA signatures in:
  - X.509 certificates (`ca_impl_openssl.py`)
  - TPM attestation quotes (`tpm_util.py`)
  - IMA file signatures (`file_signatures.py`)
  - DSSE envelope signatures (`dsse/ecdsa.py`)
- **Advantages**: Fastest PQC signature scheme, good signature sizes
- **Challenges**: Large public keys/signatures vs classical algorithms

**SLH-DSA (Stateless Hash-Based Signature Algorithm, formerly SPHINCS+)**

- **Standard**: FIPS 205
- **Use Cases**: High-security scenarios, long-term signatures, CA root certificates
- **Parameter Sets**:
  - SLH-DSA-128f: ~32 bytes public key, ~17KB signature (fast variant)
  - SLH-DSA-128s: ~32 bytes public key, ~7.9KB signature (small variant)
- **Advantages**: Based on well-understood hash functions, conservative security
- **Disadvantages**: Very large signatures
- **Recommendation**: Use for CA root certificates where long-term security is paramount

**For Key Encapsulation/Exchange (Replace RSA/ECDH):**

**ML-KEM (Module-Lattice-Based Key Encapsulation Mechanism, formerly Kyber)**

- **Standard**: FIPS 203
- **Use Cases**: TLS key exchange, encrypted payload delivery, TPM credential encryption
- **Parameter Sets**:
  - ML-KEM-512: 800 bytes public key, 768 bytes ciphertext (128-bit security)
  - ML-KEM-768: 1184 bytes public key, 1088 bytes ciphertext (192-bit security) ✅ **Recommended**
  - ML-KEM-1024: 1568 bytes public key, 1568 bytes ciphertext (256-bit security)
- **Impact**: Replace RSA encryption in:
  - Payload encryption (`crypto.py:rsa_encrypt`)
  - TPM credential blobs (`tpm_util.py`)
  - TLS handshakes (`web_util.py`)
- **Advantages**: Fast, relatively compact ciphertexts
- **Performance**: ~50-100µs for encapsulation/decapsulation

#### 3. Implement Hybrid Cryptography (Classical + PQC)

- **Rationale**: Hedges against potential PQC algorithm breaks while providing quantum resistance
- **Approach**: Combine classical algorithms (RSA/ECDSA) with PQC algorithms
- **Strategy**: Security if **either** algorithm remains unbroken

**Hybrid Key Encapsulation:**
```
Hybrid-KEM = ML-KEM-768 || ECDH-P256
shared_secret = KDF(ml_kem_secret || ecdh_secret)
```

**Hybrid Signatures:**
```
hybrid_signature = {
  "classical": ecdsa_sign(message),
  "pqc": ml_dsa_sign(message)
}
verify = classical_verify(sig.classical, msg) AND pqc_verify(sig.pqc, msg)
```

**Benefits:**
- Security even if one algorithm is compromised
- Gradual transition path
- Backwards compatibility with classical-only systems

**Implementation Locations:**
- `crypto.py`: Add hybrid encryption/signature functions
- `ca_impl_openssl.py`: Support hybrid X.509 certificates
- `signing.py`: Hybrid DSSE signatures

**Timeline**: Phase 1 (Q4 2026 - Q2 2027)

#### 4. Update TLS to Support Post-Quantum Cipher Suites

- **Current State**: TLS 1.2+ with classical ECDHE/DHE
- **Target State**: TLS 1.3 with hybrid or pure PQC key exchange

**Recommended Cipher Suites:**

1. **Hybrid Key Exchange:**
   - `X25519MLKEM768`: Classical ECDH (X25519) combined with ML-KEM-768
   - Provides quantum resistance while maintaining classical security

2. **Pure PQC Key Exchange:**
   - `MLKEM768`: Pure ML-KEM-768 for maximum quantum resistance

3. **Symmetric Encryption** (already quantum-safe):
   - `TLS_AES_256_GCM_SHA384`: AES-256-GCM with SHA-384
   - `TLS_CHACHA20_POLY1305_SHA256`: Alternative AEAD cipher

**Implementation Actions:**

1. Update `web_util.py:generate_tls_context()` to configure PQC cipher suites
2. Test with OpenSSL 3.2+ or BoringSSL with PQC support
3. Integrate OQS-Provider (Open Quantum Safe) for PQC TLS
4. Update tornado/Python ssl module dependencies
5. Add configuration options for cipher suite selection

**Example Configuration:**
```python
# web_util.py
context = ssl.create_default_context(ssl_purpose)
context.minimum_version = ssl.TLSVersion.TLSv1_3
# Enable hybrid PQC cipher suites (when available)
context.set_ciphers('X25519MLKEM768:MLKEM768:TLS_AES_256_GCM_SHA384')
```

**Dependencies:**
- OpenSSL 3.2+ with OQS-Provider
- liboqs (Open Quantum Safe library)
- Python 3.10+ for TLS 1.3 full support

**Timeline**: Q3 2026 (when library support matures)

### HIGH PRIORITY (P1) - Immediate Security Improvements

#### 5. Increase Key Sizes for Classical Algorithms (Interim Measure)

While migrating to PQC, increase classical key sizes for better quantum resistance delay:

**RSA Key Size Upgrade:**
- **Current Default**: 2048 bits
- **Recommended**: 4096 bits
- **Locations**:
  - `ca_impl_openssl.py`: Update default `cert_bits` configuration
  - Configuration files: `/etc/keylime/ca.conf`
- **Benefit**: Extends time until quantum computers can break (Mosca's theorem)
- **Trade-off**: ~4x slower key generation and signature operations
- **Timeline**: Immediate (configuration change)

**ECC Curve Upgrade:**
- **Current Default**: P-256 (SECP256R1)
- **Recommended Default**: P-384
- **Locations**:
  - `common/algorithms.py`: Update default ECC curve
  - `dsse/ecdsa.py`: Change from SECP256R1 to SECP384R1
- **Benefit**: 192-bit quantum security vs 128-bit
- **Trade-off**: Minimal performance impact
- **Timeline**: Next minor release

#### 6. Eliminate SHA-1 Usage

- **Current Usage**:
  - RSA OAEP padding (`crypto.py:138-145`)
  - IMA legacy support (`file_signatures.py`)
  - TPM operations (legacy compatibility)
- **Security Issue**: Vulnerable to collision attacks (classical), weak quantum resistance
- **Replacement**: SHA-256 (minimum) or SHA-384 (preferred)

**Migration Path:**
1. **Phase 1** (Q2 2026): Add deprecation warnings for SHA-1 usage
2. **Phase 2** (Q3 2026): Change default to SHA-256, maintain SHA-1 with explicit opt-in
3. **Phase 3** (Q4 2026): Remove SHA-1 support entirely

**Impact Analysis:**
- Breaking change for legacy IMA policies using SHA-1
- Requires TPM firmware update coordination
- Update documentation and migration guide

**Implementation:**
- `crypto.py`: Replace SHA1 with SHA256 in OAEP padding
- `file_signatures.py`: Remove SHA-1 from supported hash algorithms
- `common/algorithms.py`: Remove Hash.SHA1 enum value

#### 7. Implement Crypto-Agility Framework

**Goal**: Make cryptographic algorithm selection configurable, swappable, and version-aware

**Design Pattern:**

```python
# keylime/crypto_providers.py (new module)

from abc import ABC, abstractmethod
from typing import Tuple

class SignatureProvider(ABC):
    """Abstract base class for signature providers"""

    @abstractmethod
    def generate_keypair(self) -> Tuple[bytes, bytes]:
        """Generate public and private key pair"""
        pass

    @abstractmethod
    def sign(self, private_key: bytes, message: bytes) -> bytes:
        """Sign a message"""
        pass

    @abstractmethod
    def verify(self, public_key: bytes, message: bytes, signature: bytes) -> bool:
        """Verify a signature"""
        pass

    @abstractmethod
    def algorithm_identifier(self) -> str:
        """Return algorithm identifier (e.g., 'rsa-pss-2048', 'ml-dsa-65')"""
        pass


class RSAPSSProvider(SignatureProvider):
    """RSA-PSS signature provider"""

    def __init__(self, key_size: int = 2048):
        self.key_size = key_size

    def generate_keypair(self) -> Tuple[bytes, bytes]:
        # Existing RSA key generation logic
        pass

    def sign(self, private_key: bytes, message: bytes) -> bytes:
        # Existing RSA-PSS signature logic
        pass

    def verify(self, public_key: bytes, message: bytes, signature: bytes) -> bool:
        # Existing verification logic
        pass

    def algorithm_identifier(self) -> str:
        return f"rsa-pss-{self.key_size}"


class ECDSAProvider(SignatureProvider):
    """ECDSA signature provider"""
    # Similar implementation
    pass


class MLDSAProvider(SignatureProvider):
    """ML-DSA (Dilithium) signature provider - Future PQC"""

    def __init__(self, security_level: int = 65):
        import oqs  # liboqs-python
        self.oqs_sig = oqs.Signature(f"ML-DSA-{security_level}")

    def generate_keypair(self) -> Tuple[bytes, bytes]:
        private_key = self.oqs_sig.generate_keypair()
        public_key = self.oqs_sig.export_public_key()
        return public_key, private_key

    def sign(self, private_key: bytes, message: bytes) -> bytes:
        return self.oqs_sig.sign(message)

    def verify(self, public_key: bytes, message: bytes, signature: bytes) -> bool:
        return self.oqs_sig.verify(message, signature, public_key)

    def algorithm_identifier(self) -> str:
        return "ml-dsa-65"


class HybridSignatureProvider(SignatureProvider):
    """Hybrid signature combining classical and PQC"""

    def __init__(self, classical: SignatureProvider, pqc: SignatureProvider):
        self.classical = classical
        self.pqc = pqc

    def sign(self, private_key: bytes, message: bytes) -> bytes:
        # private_key is a composite of classical and PQC keys
        classical_key, pqc_key = self._split_key(private_key)
        sig1 = self.classical.sign(classical_key, message)
        sig2 = self.pqc.sign(pqc_key, message)
        return self._combine_signatures(sig1, sig2)

    def verify(self, public_key: bytes, message: bytes, signature: bytes) -> bool:
        classical_key, pqc_key = self._split_key(public_key)
        sig1, sig2 = self._split_signature(signature)
        # Both must verify for hybrid signature to be valid
        return (self.classical.verify(classical_key, message, sig1) and
                self.pqc.verify(pqc_key, message, sig2))


# Provider registry
class CryptoRegistry:
    """Registry for crypto providers with configuration support"""

    _providers = {}

    @classmethod
    def register(cls, name: str, provider: SignatureProvider):
        cls._providers[name] = provider

    @classmethod
    def get(cls, name: str) -> SignatureProvider:
        return cls._providers.get(name)

    @classmethod
    def from_config(cls, component: str) -> SignatureProvider:
        """Load provider from configuration"""
        algorithm = config.get(component, "signature_algorithm", fallback="rsa-pss-2048")
        return cls.get(algorithm)


# Initialize providers
CryptoRegistry.register("rsa-pss-2048", RSAPSSProvider(2048))
CryptoRegistry.register("rsa-pss-4096", RSAPSSProvider(4096))
CryptoRegistry.register("ecdsa-p256", ECDSAProvider("P256"))
CryptoRegistry.register("ecdsa-p384", ECDSAProvider("P384"))
CryptoRegistry.register("ml-dsa-65", MLDSAProvider(65))
CryptoRegistry.register("hybrid-ecdsa-mldsa",
                       HybridSignatureProvider(ECDSAProvider("P256"), MLDSAProvider(65)))
```

**Benefits:**
- Easy algorithm migration via configuration
- A/B testing of PQC algorithms
- Rollback capability if issues discovered
- Clean separation of concerns

**Files to Refactor:**
- Create new: `keylime/crypto_providers.py`
- Update: `crypto.py`, `ca_impl_openssl.py`, `cert_utils.py`, `signing.py`
- Add configuration: `/etc/keylime/*.conf` with `signature_algorithm` option

**Timeline**: Q3-Q4 2026

#### 8. Add PQC Algorithm Negotiation to API

**Current State**: Algorithm selection is hardcoded or config-based

**Proposed Enhancement**: Add algorithm negotiation to verifier-agent-registrar protocol

**Protocol Extension:**

```json
// Agent registration with supported algorithms
{
  "agent_id": "d432fbb3-d2f1-4a97-9ef7-75bd81c00000",
  "aik": "...",
  "supported_algorithms": {
    "signature": ["ml-dsa-65", "hybrid-ecdsa-mldsa", "ecdsa-p256", "rsa-pss-2048"],
    "kem": ["ml-kem-768", "hybrid-ecdh-mlkem", "ecdh-p256", "rsa-2048"],
    "hash": ["sha384", "sha256"],
    "priority": "quantum-safe"  // or "performance" or "compatibility"
  }
}

// Verifier response with selected algorithms
{
  "status": "registered",
  "selected_algorithms": {
    "signature": "hybrid-ecdsa-mldsa",
    "kem": "hybrid-ecdh-mlkem",
    "hash": "sha384"
  }
}
```

**Implementation:**
1. Add algorithm capabilities to agent registration (`models/registrar/registrar_agent.py`)
2. Implement algorithm negotiation logic in registrar (`web/registrar_server.py`)
3. Update tenant to respect negotiated algorithms (`tenant.py`)
4. Add algorithm metadata to attestation quotes

**Benefits:**
- Gradual rollout of PQC algorithms
- Backwards compatibility with legacy agents
- Allows heterogeneous deployments
- Operator control over security/performance trade-offs

**Timeline**: Q4 2026

### MEDIUM PRIORITY (P2) - Infrastructure & Tooling

#### 9. Integrate liboqs (Open Quantum Safe) Library

**Library**: https://github.com/open-quantum-safe/liboqs
**Python Bindings**: https://github.com/open-quantum-safe/liboqs-python

**Algorithms Provided:**
- **KEMs**: ML-KEM (Kyber), BIKE, Classic McEliece, HQC, FrodoKEM
- **Signatures**: ML-DSA (Dilithium), SLH-DSA (SPHINCS+), Falcon
- **Status**: Implements NIST PQC standards

**Integration Steps:**

1. **Add Dependency:**
```python
# requirements.txt
liboqs-python>=0.10.0  # MIT License
```

2. **Create PQC Wrapper Module:**
```python
# keylime/pqc.py (new module)

"""
Post-Quantum Cryptography wrapper using liboqs
"""

import oqs
from typing import Tuple, Optional

class PQCSignature:
    """Wrapper for liboqs signature algorithms"""

    SUPPORTED_ALGORITHMS = {
        "ml-dsa-44": "ML-DSA-44",
        "ml-dsa-65": "ML-DSA-65",
        "ml-dsa-87": "ML-DSA-87",
        "slh-dsa-128f": "SLH-DSA-128f",
        "slh-dsa-128s": "SLH-DSA-128s",
    }

    def __init__(self, algorithm: str = "ml-dsa-65"):
        if algorithm not in self.SUPPORTED_ALGORITHMS:
            raise ValueError(f"Unsupported algorithm: {algorithm}")

        oqs_name = self.SUPPORTED_ALGORITHMS[algorithm]
        self.sig = oqs.Signature(oqs_name)
        self.algorithm = algorithm

    def generate_keypair(self) -> Tuple[bytes, bytes]:
        """Generate public and private key pair"""
        private_key = self.sig.generate_keypair()
        public_key = self.sig.export_public_key()
        return public_key, private_key

    def sign(self, message: bytes, private_key: Optional[bytes] = None) -> bytes:
        """Sign a message with private key"""
        return self.sig.sign(message)

    def verify(self, message: bytes, signature: bytes, public_key: bytes) -> bool:
        """Verify signature"""
        try:
            return self.sig.verify(message, signature, public_key)
        except Exception:
            return False

    @property
    def public_key_length(self) -> int:
        return self.sig.details['length_public_key']

    @property
    def signature_length(self) -> int:
        return self.sig.details['length_signature']


class PQCKEM:
    """Wrapper for liboqs KEM algorithms"""

    SUPPORTED_ALGORITHMS = {
        "ml-kem-512": "ML-KEM-512",
        "ml-kem-768": "ML-KEM-768",
        "ml-kem-1024": "ML-KEM-1024",
    }

    def __init__(self, algorithm: str = "ml-kem-768"):
        if algorithm not in self.SUPPORTED_ALGORITHMS:
            raise ValueError(f"Unsupported algorithm: {algorithm}")

        oqs_name = self.SUPPORTED_ALGORITHMS[algorithm]
        self.kem = oqs.KeyEncapsulation(oqs_name)
        self.algorithm = algorithm

    def generate_keypair(self) -> Tuple[bytes, bytes]:
        """Generate public and private key pair"""
        private_key = self.kem.generate_keypair()
        public_key = self.kem.export_public_key()
        return public_key, private_key

    def encapsulate(self, public_key: bytes) -> Tuple[bytes, bytes]:
        """Encapsulate shared secret with public key"""
        ciphertext, shared_secret = self.kem.encap_secret(public_key)
        return ciphertext, shared_secret

    def decapsulate(self, ciphertext: bytes, private_key: Optional[bytes] = None) -> bytes:
        """Decapsulate shared secret with private key"""
        return self.kem.decap_secret(ciphertext)

    @property
    def ciphertext_length(self) -> int:
        return self.kem.details['length_ciphertext']

    @property
    def shared_secret_length(self) -> int:
        return self.kem.details['length_shared_secret']
```

3. **Update crypto.py to use PQC:**
```python
# keylime/crypto.py

from keylime.pqc import PQCKEM, PQCSignature

def pqc_encrypt(plaintext: bytes, public_key: bytes, algorithm: str = "ml-kem-768") -> bytes:
    """Encrypt using post-quantum KEM"""
    kem = PQCKEM(algorithm)
    ciphertext_kem, shared_secret = kem.encapsulate(public_key)

    # Use shared secret to encrypt plaintext with AES-256-GCM
    aes_ciphertext = encrypt(plaintext, shared_secret[:32])  # Use first 32 bytes as AES key

    # Combine KEM ciphertext and AES ciphertext
    return ciphertext_kem + aes_ciphertext

def pqc_decrypt(ciphertext: bytes, private_key: bytes, algorithm: str = "ml-kem-768") -> bytes:
    """Decrypt using post-quantum KEM"""
    kem = PQCKEM(algorithm)

    # Split combined ciphertext
    kem_ct_len = kem.ciphertext_length
    ciphertext_kem = ciphertext[:kem_ct_len]
    aes_ciphertext = ciphertext[kem_ct_len:]

    # Decapsulate to get shared secret
    shared_secret = kem.decapsulate(ciphertext_kem, private_key)

    # Decrypt AES ciphertext
    return decrypt(aes_ciphertext, shared_secret[:32])
```

**Testing:**
- Create `test/test_pqc.py` for unit tests
- Benchmark performance vs classical algorithms
- Integration tests with existing Keylime components

**Timeline**: Q3 2026

#### 10. Update TPM 2.0 Integration for PQC

**Challenge**: TPM 2.0 specification (2014) predates PQC standardization

**Current TPM Algorithms:**
- Asymmetric: RSA (1024-4096), ECC (P-192 through P-521)
- Hash: SHA-1, SHA-256, SHA-384, SHA-512
- Symmetric: AES (128, 192, 256), SM4

**TCG PQC Status**:
- TCG is developing TPM 2.0 Library Specification updates for PQC
- Expected timeline: 2026-2027 for specification, 2028+ for hardware

**Short-Term Approach**: Software PQC Layer Above TPM

**Architecture:**
```
┌─────────────────────────────────────┐
│  Keylime Verifier/Registrar         │
├─────────────────────────────────────┤
│  PQC Attestation Layer              │
│  - Verify PQC signatures            │
│  - Verify classical TPM signatures  │
│  - Validate binding between layers  │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│  Keylime Agent                      │
├─────────────────────────────────────┤
│  PQC Software Module                │
│  - ML-DSA keypair in software       │
│  - Sign TPM quotes with ML-DSA      │
│  - Bind PQC key to TPM AIK          │
├─────────────────────────────────────┤
│  TPM 2.0 (Classical)                │
│  - Generate quotes with ECDSA/RSA   │
│  - Provide hardware root of trust   │
│  - Seal PQC private key             │
└─────────────────────────────────────┘
```

**Hybrid Attestation Protocol:**

1. **Key Binding**: Bind software PQC key to hardware TPM AIK
   ```
   pqc_public_key = generate_ml_dsa_keypair()
   pqc_binding_cert = tpm_sign(pqc_public_key)  // Classical TPM signature
   ```

2. **Quote Generation**: Produce dual signatures
   ```
   quote = {
     "pcrs": {...},
     "nonce": "...",
     "tpm_signature": tpm_sign_ecdsa(pcrs || nonce),  // Hardware TPM
     "pqc_signature": ml_dsa_sign(pcrs || nonce),     // Software PQC
     "binding_cert": pqc_binding_cert
   }
   ```

3. **Verification**: Verify both signatures
   ```
   verify_tpm_signature(quote.tpm_signature, aik_public)  // Classical verification
   verify_pqc_signature(quote.pqc_signature, pqc_public)  // PQC verification
   verify_binding(pqc_public, binding_cert, aik_public)   // Binding validation
   ```

**Benefits:**
- Maintains hardware root of trust via TPM
- Adds quantum resistance via software PQC
- Binding ensures PQC key authenticity
- Gradual transition path to hardware PQC TPMs

**Implementation:**
- Update `tpm/tpm_util.py` with hybrid attestation functions
- Store PQC keys in TPM NVRAM (encrypted with TPM)
- Seal PQC private key to PCR values

**Medium-Term**: Adopt TCG PQC TPM Standard (2027-2028)
- Monitor TCG specification updates
- Test with pre-release PQC TPM modules
- Update software to support hardware PQC operations

**Long-Term**: Pure PQC TPM Attestation (2029+)
- Transition to hardware PQC TPMs
- Deprecate classical-only attestation
- Maintain hybrid mode for legacy support

**Timeline**:
- Software hybrid: Q4 2026 - Q1 2027
- TCG standard monitoring: Ongoing
- Hardware PQC TPM: 2028+

#### 11. Certificate Chain PQC Migration

**Challenge**: X.509 PKI infrastructure assumes RSA/ECDSA signatures

**Relevant Standards:**
- **RFC 8391**: XMSS (hash-based signatures) in X.509
- **Draft IETF LAMPS**: ML-DSA in X.509 certificates (draft-ietf-lamps-dilithium-certificates)
- **Composite Signatures**: Classical + PQC in single certificate
- **Alternative Certificates**: Separate classical and PQC certificate chains

**Migration Path:**

**Phase 1: Hybrid Certificates (2026-2027)**

Generate certificates with dual signatures:

```
Certificate Structure:
├─ Subject: CN=Keylime Agent
├─ Public Key: Composite (ECDSA-P256 || ML-DSA-65)
├─ Signature Algorithm: Composite-ECDSA-MLDSA
├─ Signature: {
│    classical: ECDSA-P256 signature over TBSCertificate,
│    pqc: ML-DSA-65 signature over TBSCertificate
│  }
└─ Issuer: CN=Keylime CA
```

**Implementation** (`ca_impl_openssl.py`):

```python
def create_hybrid_certificate(subject, public_key_classical, public_key_pqc, issuer_key, ...):
    """Create X.509 certificate with hybrid signatures"""

    # Create TBS (To-Be-Signed) certificate
    cert_builder = x509.CertificateBuilder()
    cert_builder = cert_builder.subject_name(subject)
    cert_builder = cert_builder.issuer_name(issuer)
    # ... other fields

    # Composite public key (encode both keys)
    composite_pubkey = encode_composite_key(public_key_classical, public_key_pqc)
    cert_builder = cert_builder.public_key(composite_pubkey)

    # Create TBS certificate bytes
    tbs_cert = cert_builder.tbs_certificate_bytes()

    # Sign with both algorithms
    classical_sig = sign_ecdsa(tbs_cert, issuer_key_classical)
    pqc_sig = sign_ml_dsa(tbs_cert, issuer_key_pqc)

    # Combine signatures into composite signature
    composite_sig = encode_composite_signature(classical_sig, pqc_sig)

    # Build final certificate with composite signature
    # (Requires custom encoding as X.509 doesn't natively support this yet)
    return build_certificate(tbs_cert, composite_sig)
```

**Verification**:
```python
def verify_hybrid_certificate(cert, ca_cert_classical, ca_cert_pqc):
    """Verify hybrid certificate - both signatures must be valid"""
    tbs_cert = extract_tbs(cert)
    classical_sig, pqc_sig = extract_composite_signature(cert)

    # Both must verify
    valid_classical = verify_ecdsa(tbs_cert, classical_sig, ca_cert_classical.public_key)
    valid_pqc = verify_ml_dsa(tbs_cert, pqc_sig, ca_cert_pqc.public_key)

    return valid_classical and valid_pqc
```

**Phase 2: Pure PQC Intermediate CAs (2027-2028)**

Create intermediate CAs using pure PQC signatures:

```
Root CA (Hybrid ECDSA+ML-DSA)
  └── Intermediate CA (Pure ML-DSA-65)
        └── End Entity Cert (Pure ML-DSA-65)
```

**Phase 3: Pure PQC Root CA (2029+)**

Transition to pure PQC root CA:
- Issue new PQC root certificate
- Distribute to all verifiers/agents/tenants
- Maintain hybrid root for backwards compatibility
- Eventually deprecate classical root

**Trust Anchor Update Process:**
1. Generate new PQC root CA certificate
2. Distribute via secure channel (out-of-band)
3. Update trust stores on all Keylime components
4. Verify all components trust new root
5. Begin issuing certificates from PQC root

**Implementation Modules:**
- `ca_impl_openssl.py`: Hybrid certificate generation
- `cert_utils.py`: Hybrid certificate parsing and verification
- `ca_util.py`: Trust store management for hybrid certificates

**Timeline**:
- Phase 1 (Hybrid): Q4 2026 - Q2 2027
- Phase 2 (PQC Intermediate): Q3 2027 - Q2 2028
- Phase 3 (PQC Root): Q3 2028+

#### 12. Performance Testing & Optimization

**Concern**: PQC algorithms have different performance characteristics than classical algorithms

**ML-DSA-65 Performance (Typical):**
- Key generation: 50-200 µs
- Signing: 100-500 µs
- Verification: 50-200 µs
- **Overhead vs ECDSA-P256**: 2-5x slower
- **Public key size**: 4032 bytes (vs 91 bytes for ECDSA-P256)
- **Signature size**: 3309 bytes (vs 64-72 bytes for ECDSA-P256)

**ML-KEM-768 Performance (Typical):**
- Key generation: 20-50 µs
- Encapsulation: 30-80 µs
- Decapsulation: 30-80 µs
- **Overhead vs ECDH-P256**: 1.5-3x slower
- **Public key size**: 1184 bytes (vs 91 bytes for ECDH-P256)
- **Ciphertext size**: 1088 bytes (vs ~91 bytes for ECDH-P256)

**Performance Testing Plan:**

1. **Benchmark Current Operations:**
   - Measure baseline performance of all crypto operations
   - Identify bottlenecks in current implementation
   - Profile agent registration, attestation, and provisioning workflows

2. **Benchmark PQC Operations:**
   - Measure ML-DSA and ML-KEM performance on target hardware
   - Test with different security levels (512, 768, 1024 for ML-KEM)
   - Profile memory usage

3. **Integration Performance Testing:**
   - End-to-end latency for agent registration with PQC
   - TPM quote verification with hybrid signatures
   - TLS handshake performance with ML-KEM
   - Scalability testing: 1k, 10k, 100k agents

**Key Bottlenecks to Monitor:**
- **TPM Quote Verification**: Verifier may need to verify thousands of quotes/second
- **TLS Handshakes**: Larger PQC keys increase handshake time
- **Certificate Chain Verification**: Hybrid certificates require double verification
- **Network Bandwidth**: Larger signatures/keys increase network traffic

**Optimization Strategies:**

1. **Hardware Acceleration:**
   - Use AVX2/AVX-512 instructions for lattice operations
   - Leverage AES-NI for symmetric operations
   - Consider FPGA/ASIC acceleration for high-throughput scenarios

2. **Algorithm Parameter Selection:**
   - Use ML-KEM-512 instead of ML-KEM-768 where 128-bit security suffices
   - Consider ML-DSA-44 for less critical signatures

3. **Caching:**
   - Cache verified PQC public keys
   - Cache certificate chain verification results
   - Implement signature batch verification where possible

4. **Batching:**
   - Batch ML-DSA signature verifications
   - Parallel processing of attestation quotes

**Performance Test Suite:**

```python
# test/test_pqc_performance.py

import time
from keylime.pqc import PQCSignature, PQCKEM
from keylime.crypto import generate_rsa_keypair, rsa_sign, rsa_verify

def benchmark_signature_scheme(scheme_name, keygen_fn, sign_fn, verify_fn, iterations=1000):
    """Benchmark signature scheme operations"""

    # Key generation
    start = time.perf_counter()
    for _ in range(iterations):
        pubkey, privkey = keygen_fn()
    keygen_time = (time.perf_counter() - start) / iterations

    # Signing
    message = b"Test message for benchmarking"
    pubkey, privkey = keygen_fn()
    start = time.perf_counter()
    for _ in range(iterations):
        signature = sign_fn(privkey, message)
    sign_time = (time.perf_counter() - start) / iterations

    # Verification
    signature = sign_fn(privkey, message)
    start = time.perf_counter()
    for _ in range(iterations):
        verify_fn(pubkey, message, signature)
    verify_time = (time.perf_counter() - start) / iterations

    print(f"{scheme_name}:")
    print(f"  Key Generation: {keygen_time*1e6:.2f} µs")
    print(f"  Signing:        {sign_time*1e6:.2f} µs")
    print(f"  Verification:   {verify_time*1e6:.2f} µs")

# Run benchmarks
benchmark_rsa_2048()
benchmark_ecdsa_p256()
benchmark_ml_dsa_65()
benchmark_hybrid()
```

**Deliverables:**
- Performance benchmark suite
- Performance comparison report (classical vs PQC vs hybrid)
- Optimization recommendations
- Scalability analysis

**Timeline**: Q3-Q4 2026

### LOW PRIORITY (P3) - Long-term Planning

#### 13. Quantum-Safe Backup & Archive

**Threat Model**: "Harvest Now, Decrypt Later" Attacks

Adversaries can:
1. Capture encrypted data today (TPM-encrypted payloads, TLS traffic, backups)
2. Store it indefinitely
3. Decrypt it when quantum computers become available

**At-Risk Data in Keylime:**
- Encrypted payloads delivered to agents
- TLS-encrypted communications (if recorded)
- Encrypted private keys in certificate archives
- Registrar database backups with sensitive data

**Mitigation Strategies:**

1. **Re-encrypt Existing Archives:**
   ```python
   # keylime/cmd/reencrypt_archives.py (new utility)

   def reencrypt_with_pqc(archive_path, output_path):
       """Re-encrypt archive with PQC KEM"""
       # Decrypt with classical RSA
       classical_data = rsa_decrypt(read_archive(archive_path), old_rsa_key)

       # Re-encrypt with ML-KEM
       pqc_encrypted = pqc_encrypt(classical_data, new_pqc_key, algorithm="ml-kem-768")

       # Write to new archive
       write_archive(output_path, pqc_encrypted)
   ```

2. **Update Backup Encryption:**
   - Update `ca_util.py:390-391` (private key encryption) to use PQC KEM
   - Encrypt database backups with ML-KEM instead of RSA
   - Use hybrid encryption for backwards compatibility

3. **Payload Encryption Migration:**
   - Update payload encryption in `tenant.py:580, 652`
   - Use ML-KEM for key encapsulation instead of RSA
   - Maintain ability to decrypt old RSA-encrypted payloads

4. **Implement Forward Secrecy:**
   - Use ephemeral PQC keys for each session
   - Don't reuse long-term PQC keys for payload encryption
   - Implement key rotation policies

**Implementation Priority:**
- High-value data: Immediate re-encryption
- Long-term archives (>10 year retention): High priority
- Short-term data (<1 year): Lower priority

**Timeline**: Q1-Q2 2027

#### 14. Quantum Random Number Generation (QRNG)

**Current State**: `secrets.token_bytes()` uses OS CSPRNG (e.g., `/dev/urandom`)

**Quantum RNG Benefits:**
- True physical randomness from quantum processes
- Not vulnerable to algorithmic biases
- Enhanced entropy for critical cryptographic operations

**Quantum RNG Technologies:**
- **ID Quantique**: Commercial QRNG devices (USB, PCIe)
- **Quintessence Labs**: High-throughput QRNG
- **Whitewood**: Entropy-as-a-Service
- **PicoQuant**: Research-grade QRNG

**Use Cases in Keylime:**
- TPM credential challenge generation
- Session token generation
- Key generation (RSA, ECC, PQC)
- Nonce generation for attestation

**Integration Approach:**

```python
# keylime/qrng.py (new module)

from typing import Optional
import secrets

class QRNGProvider:
    """Abstract QRNG provider interface"""

    def get_random_bytes(self, num_bytes: int) -> bytes:
        raise NotImplementedError

class SystemQRNG(QRNGProvider):
    """Fallback to system CSPRNG"""
    def get_random_bytes(self, num_bytes: int) -> bytes:
        return secrets.token_bytes(num_bytes)

class IDQuantiqueQRNG(QRNGProvider):
    """ID Quantique QRNG device"""
    def __init__(self, device_path: str = "/dev/qrng"):
        self.device = open(device_path, "rb")

    def get_random_bytes(self, num_bytes: int) -> bytes:
        return self.device.read(num_bytes)

# Global QRNG instance with fallback
_qrng: Optional[QRNGProvider] = None

def init_qrng(provider: str = "system"):
    """Initialize QRNG provider"""
    global _qrng
    if provider == "idq":
        _qrng = IDQuantiqueQRNG()
    else:
        _qrng = SystemQRNG()

def get_random_bytes(num_bytes: int) -> bytes:
    """Get random bytes from configured QRNG"""
    if _qrng is None:
        init_qrng()
    return _qrng.get_random_bytes(num_bytes)
```

**Assessment:**
- **Priority**: Low - classical CSPRNGs remain secure against quantum attacks
- **Cost**: QRNG devices range from $500 (USB) to $10k+ (high-throughput)
- **Benefit**: Marginal security improvement; mostly relevant for paranoid threat models

**Recommendation**:
- Not critical for quantum-readiness
- Consider for high-security deployments
- Monitor commodity QRNG availability

**Timeline**: 2028+ (optional enhancement)

#### 15. Monitor NIST PQC Standardization Updates

**Current Status** (March 2026):
- **Finalized Standards** (2024):
  - FIPS 203: ML-KEM (Kyber)
  - FIPS 204: ML-DSA (Dilithium)
  - FIPS 205: SLH-DSA (SPHINCS+)
- **Ongoing** (Round 4):
  - Additional signature schemes: Falcon
  - Additional KEMs: BIKE, Classic McEliece, HQC

**Why Monitor:**
- Algorithm updates and parameter changes
- Discovery of vulnerabilities or side-channel attacks
- New algorithms may be standardized
- Implementation guidance and best practices

**What to Track:**

1. **NIST Updates:**
   - FIPS standard revisions
   - NIST Special Publications (SP 800 series) for PQC guidance
   - NIST Cryptographic Algorithm Validation Program (CAVP) for PQC

2. **Academic Research:**
   - Side-channel attacks on PQC implementations
   - Cryptanalysis of PQC algorithms
   - Performance optimizations

3. **Standards Bodies:**
   - IETF (Internet Engineering Task Force): TLS 1.3 PQC cipher suites
   - TCG (Trusted Computing Group): TPM PQC support
   - ETSI (European): PQC standards for telecommunications

4. **Implementation Security:**
   - Constant-time implementation requirements
   - Fault injection attacks
   - Power analysis vulnerabilities

**Action Items:**

- Subscribe to NIST PQC mailing list: pqc-forum@list.nist.gov
- Monitor NIST PQC project page: https://csrc.nist.gov/projects/post-quantum-cryptography
- Track CVEs related to PQC implementations
- Participate in PQC migration working groups

**Risk Mitigation:**
- Crypto-agility framework enables rapid algorithm updates
- Hybrid approach provides safety net if PQC algorithm breaks
- Regular security audits of PQC implementations

**Timeline**: Ongoing (continuous monitoring)

#### 16. Collaborate with TPM/TCG Community

**Rationale**: Keylime's security model depends on TPM, which is specified by Trusted Computing Group (TCG)

**TCG PQC Activities:**
- TCG working on TPM 2.0 Library Specification updates for PQC
- Expected algorithms: ML-DSA, ML-KEM (possibly Falcon, FrodoKEM)
- Timeline: Specification in 2026-2027, hardware in 2028+

**Collaboration Opportunities:**

1. **Requirements Input:**
   - Provide Keylime use cases to TCG PQC working group
   - Identify attestation-specific PQC requirements
   - Advocate for algorithm flexibility

2. **Early Testing:**
   - Test with pre-release PQC TPM simulators
   - Provide feedback on API design
   - Validate performance characteristics

3. **Standards Development:**
   - Participate in TCG attestation specifications
   - Contribute to remote attestation PQC profiles
   - Ensure compatibility with Keylime architecture

4. **Community Engagement:**
   - Present Keylime PQC migration at TCG meetings
   - Publish case studies and lessons learned
   - Collaborate on reference implementations

**TCG Contacts:**
- TCG website: https://trustedcomputinggroup.org/
- TPM Working Group
- Attestation WG

**Keylime Contributions:**
- Software PQC layer design (can inform TCG API design)
- Performance requirements for attestation at scale
- Hybrid attestation protocol

**Benefits:**
- Early access to PQC TPM specifications
- Influence standards to meet Keylime needs
- Knowledge sharing with broader community
- Accelerate PQC TPM adoption

**Timeline**: Ongoing (2026+)

---

## Implementation Timeline

### Phase 0: Preparation (Q2-Q3 2026)

**Goals**: Establish foundation for PQC migration

**Deliverables:**
- [ ] Form PQC working group (security, architecture, TPM experts)
- [ ] Create detailed PQC migration plan and roadmap
- [ ] Add liboqs and liboqs-python to development environment
- [ ] Benchmark current cryptographic performance (baseline)
- [ ] Design and document crypto-agility abstraction layer
- [ ] Identify all cryptographic touchpoints in codebase
- [ ] Set up continuous NIST/TCG standards monitoring
- [ ] Community announcement and RFC for PQC migration

**Effort**: 2 person-months

### Phase 1: Hybrid Cryptography (Q4 2026 - Q2 2027)

**Goals**: Deploy hybrid classical+PQC cryptography for quantum resistance with backwards compatibility

**Deliverables:**
- [ ] Implement crypto-agility framework (`crypto_providers.py`)
- [ ] Integrate liboqs for ML-DSA and ML-KEM
- [ ] Implement hybrid signatures (ECDSA+ML-DSA or RSA+ML-DSA)
- [ ] Implement hybrid KEMs (ECDH+ML-KEM) for payload encryption
- [ ] Update TLS configuration to support hybrid cipher suites (requires OpenSSL 3.2+)
- [ ] Add PQC algorithm negotiation to agent registration API
- [ ] Update `crypto.py` with hybrid encryption/decryption functions
- [ ] Create hybrid X.509 certificates (`ca_impl_openssl.py`)
- [ ] Comprehensive testing (unit, integration, end-to-end)
- [ ] Performance benchmarking and optimization
- [ ] Documentation updates

**Effort**: 8-10 person-months

**Risk**: OpenSSL/library PQC support may not be mature; fallback to pure software implementation

### Phase 2: PQC Transition (Q3 2027 - Q2 2028)

**Goals**: Enable pure PQC as an option, expand deployment

**Deliverables:**
- [ ] Enable pure PQC mode (ML-DSA, ML-KEM) as configuration option
- [ ] Issue PQC-enabled X.509 certificates (intermediate CAs)
- [ ] Update IMA signature verification for PQC signatures
- [ ] Migrate DSSE to PQC signatures (ML-DSA)
- [ ] Implement software PQC layer for TPM hybrid attestation
- [ ] Re-encrypt sensitive archives with PQC KEMs
- [ ] Extensive compatibility testing (legacy agents, mixed deployments)
- [ ] Performance optimization (batch verification, caching)
- [ ] Security audit of PQC implementation
- [ ] Community testing program (beta testers)

**Effort**: 6-8 person-months

**Milestone**: First production deployment with hybrid PQC

### Phase 3: PQC by Default (Q3 2028 - Q4 2028)

**Goals**: Make PQC algorithms the default choice

**Deliverables:**
- [ ] Change default algorithms to PQC in configuration
- [ ] Deprecate pure classical algorithms (with warnings)
- [ ] Update all documentation to recommend PQC
- [ ] Community education campaign (blog posts, talks, tutorials)
- [ ] Migration guide for existing deployments
- [ ] Automated migration tools
- [ ] Monitor TCG PQC TPM specifications for hardware readiness

**Effort**: 3-4 person-months

**Milestone**: PQC becomes recommended default configuration

### Phase 4: Classical Deprecation (2029+)

**Goals**: Remove support for pure classical algorithms

**Deliverables:**
- [ ] Issue PQC root CA certificate
- [ ] Deprecation warnings for RSA/ECDSA-only configurations
- [ ] Remove pure RSA/ECDSA support (maintain hybrid for legacy)
- [ ] Pure PQC default deployment
- [ ] Adopt hardware PQC TPMs when available
- [ ] Long-term monitoring and algorithm updates

**Effort**: 2-3 person-months

**Milestone**: Pure PQC deployment, classical algorithms legacy-only

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation Strategy |
|------|-----------|--------|---------------------|
| **Quantum computer breaks RSA/ECC before migration complete** | Medium (2030-2035) | Critical - Complete compromise of attestation and encryption | Accelerate hybrid deployment to Q4 2026; prioritize high-value data re-encryption; monitor quantum computing progress via NIST/industry reports |
| **PQC algorithm vulnerability discovered** | Low | High - Need to switch algorithms rapidly | Use hybrid approach (security if either algorithm remains secure); implement crypto-agility framework for rapid algorithm swap; continuous monitoring of academic research |
| **Performance degradation from PQC unacceptable** | Medium-High | Medium - Slower attestation, higher latency | Extensive performance testing early (Phase 0); hardware acceleration (AVX2/AVX-512); algorithm parameter tuning; consider ML-KEM-512 for non-critical paths; caching and batching optimizations |
| **Incompatibility with legacy TPMs** | High | High - Cannot attest legacy hardware | Maintain classical algorithm fallback; implement software PQC layer above TPM; hybrid attestation protocol; long deprecation timeline for classical-only mode (2029+) |
| **NIST standards change PQC algorithms** | Medium | Medium - Need to reimplement | Track NIST standardization closely; crypto-agility framework; participate in standards development; hybrid approach reduces impact |
| **Library/tooling immaturity** | Medium | Medium - Implementation delays, bugs | Engage with liboqs community; contribute fixes; have fallback pure Python implementation; extensive testing; security audits |
| **Ecosystem not ready for PQC X.509 certs** | High | Medium - Interoperability issues | Use hybrid certificates; maintain classical certificate chain alongside PQC; gradual rollout; work with certificate validation libraries |
| **Increased bandwidth usage** | Low | Low - Higher network costs | Compression of PQC signatures; optimize protocol to minimize signature transmissions; acceptable trade-off for quantum security |
| **Key/signature size storage issues** | Low | Low - Database growth | Database schema updates; compression; acceptable with modern storage costs; plan for 3-5x signature size increase |
| **Community resistance to change** | Medium | Medium - Slow adoption | Education campaign; clear migration guides; automated tools; demonstrate security benefits; emphasize hybrid approach reduces risk |

---

## Estimated Effort

### Development Effort Breakdown

| Task | Effort (person-months) |
|------|------------------------|
| Crypto-agility framework design and implementation | 2-3 |
| liboqs integration and PQC wrapper module | 1-2 |
| Hybrid signature implementation (ML-DSA) | 2-3 |
| Hybrid KEM implementation (ML-KEM) | 1-2 |
| TLS PQC cipher suite support | 2-3 |
| X.509 PQC certificate generation and verification | 2-3 |
| API algorithm negotiation protocol | 1-2 |
| TPM hybrid attestation layer | 2-3 |
| Archive re-encryption utilities | 1 |
| Testing and validation (unit, integration, E2E) | 3-4 |
| Performance benchmarking and optimization | 2-3 |
| Documentation and migration guides | 1-2 |
| Security audit and review | 1-2 |

**Total Estimated Effort**: 21-35 person-months

### Team Composition

**Recommended Team** (for 12-month timeline):
- 1x Senior Cryptography Engineer (lead)
- 1x Senior Software Engineer (implementation)
- 0.5x TPM/Attestation Expert (part-time, consulting)
- 0.5x Performance Engineer (part-time, optimization)
- 0.25x Technical Writer (documentation)

**Effective Team Size**: ~2-2.5 FTEs

**Timeline**: 12-18 months for Phases 0-2 (through hybrid deployment)

### Cost Estimate

**Personnel** (12 months, 2.5 FTEs):
- Senior engineers: $150-200k/year each
- Total: ~$375-500k

**Infrastructure/Tools**:
- Development hardware: $10-20k
- Testing infrastructure: $5-10k
- PQC TPM simulators/hardware: $5-10k
- Security audit (external): $20-50k

**Total Estimated Cost**: $415-590k for 12-month Phase 0-2 delivery

---

## Conclusion

Keylime currently relies heavily on **classical public-key cryptography (RSA, ECDSA)** that is **not quantum-safe**. The project uses RSA and ECC extensively for:

- **Digital Signatures**: Attestation quotes, X.509 certificates, IMA signatures, DSSE envelopes
- **Key Encapsulation**: Encrypted payload delivery, TLS handshakes, TPM credential encryption
- **PKI Infrastructure**: Certificate chains, CRLs, trust anchors

All of these are vulnerable to **Shor's algorithm** on a large-scale quantum computer, with potential breakthrough timeline estimated around 2030-2040.

**Quantum-Resistant Components:**
- Symmetric encryption (AES-256-GCM): ✅ Acceptable (128-bit quantum security)
- Hash functions (SHA-256/384/512): ✅ Acceptable to excellent quantum resistance
- HMAC (SHA-384): ✅ Good quantum resistance

**Critical Vulnerabilities:**
- RSA encryption/signatures: ❌ Fully quantum-vulnerable
- ECC/ECDSA: ❌ Fully quantum-vulnerable
- TLS key exchange (ECDHE/DHE): ❌ Fully quantum-vulnerable

### Recommended Action Plan

**IMMEDIATE** (Q2 2026):
1. Form PQC working group
2. Increase RSA key size to 4096 bits (interim measure)
3. Eliminate SHA-1 usage
4. Begin crypto-agility framework design

**SHORT-TERM** (Q4 2026 - Q2 2027):
1. Integrate liboqs for ML-KEM and ML-DSA
2. Deploy hybrid cryptography (classical + PQC)
3. Update TLS to support PQC cipher suites
4. Implement algorithm negotiation in API

**MEDIUM-TERM** (Q3 2027 - Q2 2028):
1. Enable pure PQC mode as option
2. Issue PQC X.509 certificates
3. Implement TPM hybrid attestation
4. Re-encrypt sensitive archives

**LONG-TERM** (2029+):
1. Make PQC the default
2. Deprecate classical-only mode
3. Adopt hardware PQC TPMs
4. Remove classical algorithm support

### Success Criteria

The Keylime project will be considered **quantum-ready** when:

✅ Hybrid (classical + PQC) cryptography is deployed and tested
✅ All asymmetric operations support ML-DSA or ML-KEM
✅ TLS connections use PQC key exchange
✅ X.509 certificates support PQC signatures
✅ TPM attestation includes PQC signatures
✅ Crypto-agility framework enables algorithm switching
✅ Performance meets operational requirements
✅ Documentation and migration guides are complete
✅ Community adoption demonstrates viability

### Strategic Importance

Post-quantum cryptography migration is not optional for a security-critical project like Keylime. The "harvest now, decrypt later" threat means that:

- Data encrypted today with classical algorithms is at risk
- Attestation signatures could be forged retroactively
- Trust relationships could be compromised

**The time to act is now.** While large-scale quantum computers may be 5-15 years away, the migration to PQC will take multiple years and requires ecosystem coordination. Starting in 2026 with a target of hybrid PQC deployment by late 2026/early 2027 positions Keylime ahead of the threat curve and aligned with NIST standardization timelines.

The availability of **NIST-standardized PQC algorithms** (ML-KEM, ML-DSA, SLH-DSA) and mature open-source implementations (**liboqs**) makes this transition achievable with the roadmap outlined in this document.

---

## References

### Standards and Specifications

1. **NIST Post-Quantum Cryptography Standardization**
   - FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard
   - FIPS 204: Module-Lattice-Based Digital Signature Standard
   - FIPS 205: Stateless Hash-Based Digital Signature Standard
   - https://csrc.nist.gov/projects/post-quantum-cryptography

2. **IETF PQC Standards (Draft)**
   - draft-ietf-lamps-dilithium-certificates: ML-DSA in X.509
   - draft-ietf-tls-hybrid-design: Hybrid Key Exchange in TLS 1.3
   - https://datatracker.ietf.org/wg/lamps/documents/

3. **TCG TPM 2.0 Specifications**
   - TPM 2.0 Library Specification
   - TCG PQC Working Group (upcoming standards)
   - https://trustedcomputinggroup.org/

### Libraries and Tools

4. **Open Quantum Safe (OQS)**
   - liboqs: C library for PQC algorithms
   - liboqs-python: Python bindings
   - https://github.com/open-quantum-safe/

5. **OpenSSL PQC Support**
   - OQS-Provider for OpenSSL 3.x
   - https://github.com/open-quantum-safe/oqs-provider

### Academic Papers

6. **Shor's Algorithm**
   - Shor, P. W. (1997). "Polynomial-Time Algorithms for Prime Factorization and Discrete Logarithms on a Quantum Computer"

7. **PQC Algorithm Papers**
   - CRYSTALS-Kyber/ML-KEM specification
   - CRYSTALS-Dilithium/ML-DSA specification
   - SPHINCS+/SLH-DSA specification

### Industry Reports

8. **Quantum Computing Timeline**
   - Mosca's Theorem and PQC Migration Timeline
   - Industry quantum computing roadmaps (IBM, Google, etc.)

9. **Migration Guidance**
   - NIST Cybersecurity White Papers on PQC Migration
   - CISA PQC Readiness Guidance

---

**Document Version**: 1.0
**Last Updated**: March 4, 2026
**Next Review**: September 2026 (6-month review cycle)
**Contact**: security@keylime.groups.io
