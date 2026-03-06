# Quantum Readiness Analysis - Trustee Operator

## Direct Cryptography Usage

### Not Quantum-Safe

1. **Ed25519 Digital Signatures**
   - `internal/controller/trusteeconfig_controller.go:671` - Key generation
   - `internal/controller/crypto_helper.go:20-21` - PEM encoding functions
   - Used for KBS authentication secrets
   - **Vulnerable** to quantum attacks (Shor's algorithm)

2. **ECDSA P-256**
   - `docs/token-configuration.md:35` - Attestation token signing (256-bit)
   - **Vulnerable** to quantum attacks (Shor's algorithm)

3. **RSA-2048**
   - `tests/e2e/sample-attester-https/01-auth-secret.yaml:9` - Test certificates
   - **Vulnerable** to quantum attacks (Shor's algorithm; 2048-bit insufficient)

4. **TLS/X.509**
   - `cmd/main.go:20-21,64` - Metrics server TLS
   - `internal/controller/crypto_helper.go:21` - Certificate handling
   - Uses classical algorithms by default (RSA/ECDSA)

## Dependency Cryptography

### Not Quantum-Safe

5. **Go stdlib** (`golang.org/x/`)
   - `crypto/rand`, `crypto/tls`, `crypto/x509`
   - Standard Go TLS 1.2/1.3 with classical cipher suites

6. **Kubernetes Client Libraries**
   - `k8s.io/client-go`, `k8s.io/apiserver`
   - TLS for API communication (classical)

7. **gRPC** (`google.golang.org/grpc v1.72.1`)
   - TLS-based authentication
   - No post-quantum support

8. **OAuth2** (`golang.org/x/oauth2 v0.27.0`)
   - JWT typically uses RSA/ECDSA

## Summary

**All cryptographic functions in this codebase are NOT quantum-safe.** The project relies entirely on classical public-key cryptography (Ed25519, ECDSA, RSA) which would be broken by Shor's algorithm on a sufficiently large quantum computer. No post-quantum algorithms (Kyber, Dilithium, SPHINCS+, etc.) are currently used.

## Analysis Date
2026-03-05
