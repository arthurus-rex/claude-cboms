# Quantum Readiness Assessment: Confidential Containers

## Cryptographic Usage Locations

### **Direct Code Dependencies** (referenced projects):

1. **ocicrypt** (`github.com/containers/ocicrypt`)
   - Container image encryption (AES-256-GCM)
   - **NOT quantum-safe** (symmetric crypto considered safe with 256-bit keys, but key exchange mechanisms are vulnerable)

2. **cosign/sigstore** (`github.com/sigstore/cosign`)
   - Image signature verification (ECDSA/Ed25519)
   - **NOT quantum-safe**

3. **guest-components** (attestation-agent, image-rs, ocicrypt-rs)
   - Attestation protocols
   - Image decryption/verification
   - KBS protocol with ECDH (v0.13.0+)
   - **NOT quantum-safe** (ECDH vulnerable to quantum attacks)

4. **Trustee** (KBS, Attestation Service, RVPS)
   - Ed25519 key pairs for authentication (guides/coco-dev.md:109)
   - Key brokering/management
   - **NOT quantum-safe**

5. **Hardware TEE Attestation**:
   - Intel TDX (typically uses ECDSA P-256/P-384)
   - AMD SEV-SNP (ECDSA-based attestation)
   - Intel SGX (ECDSA-based remote attestation)
   - IBM SE/PEF (RSA/ECDSA-based)
   - TPM attestation (RSA/ECC-based)
   - **NOT quantum-safe**

6. **SSH** (demos/ssh-demo):
   - Ed25519 host/user keys
   - **NOT quantum-safe**

### **Network Security**:
- TLS for KBS communication (standard RSA/ECDHE)
- **NOT quantum-safe**

## Quantum-Safety Assessment

**NONE of the cryptographic components are quantum-safe.** All rely on:
- Elliptic Curve Cryptography (ECDSA, ECDH, Ed25519) - vulnerable to Shor's algorithm
- RSA (if present) - vulnerable to Shor's algorithm
- AES-256 symmetric encryption is quantum-resistant but key exchange uses vulnerable ECC/RSA

**Recommendation**: Migration to post-quantum cryptography (NIST standards: ML-KEM, ML-DSA, SLH-DSA) would be required for quantum resistance.
