# Web Content Authenticity Protocol (WCAP)

**Version 0.3 – Draft**
 **Status of This Memo**
 This document specifies a proposed protocol for the Internet community and requests discussion and suggestions for improvement. Distribution of this memo is unlimited.

------

## Abstract

This document defines the Web Content Authenticity Protocol (WCAP), a mechanism for ensuring the end-to-end authenticity and integrity of web resources. WCAP is designed to be transport-agnostic, meaning it can verify content regardless of how it was delivered—via HTTP, peer-to-peer networks, or other channels. By using digital signatures, WCAP allows a verifier to cryptographically confirm that a resource is exactly what the originator intended to publish, and has not been tampered with by any intermediary, including compromised servers, state-level actors, or malicious networks. This provides a powerful guarantee against censorship, manipulation, and unauthorized modification of information. Version 0.3 extends the protocol with an optional Signing Policy Declaration mechanism, allowing Signers to declare what they verified about content provenance and integrity before signing, enabling verifiers to distinguish transport-integrity-only signatures from signatures backed by deeper content verification.

------

## 1. Introduction

While Transport Layer Security (TLS) provides a secure channel between a client and a server, it does not protect content once it has been delivered or if an intermediary in the chain (such as a CDN or proxy) is compromised. The Web Content Authenticity Protocol (WCAP) provides a mechanism for a web server (the **Signer**) to digitally sign web resources, allowing a client (the **Verifier**) to confirm that the content is exactly what the origin server intended to send.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

------

## 2. Threat Model

WCAP is designed to protect against the following threats:

- **Content Injection/Modification:** A malicious actor alters content in transit.
- **Compromised Infrastructure:** A CDN, reverse proxy, or hosting server is compromised.
- **DNS Manipulation:** An adversary reroutes traffic to a malicious server.
- **TLS Interception:** A rogue or coerced CA enables a man-in-the-middle.
- **Censorship and Takedowns:** A government forces a platform to alter or remove content.
- **Signer Coercion:** A publisher is forced to hand over their private signing keys.
- **Provenance Ambiguity:** A Signer forwards content from an untrusted or unverified upstream source without disclosing this to the Verifier, causing the Verifier to incorrectly infer that the Signer vouches for the content's origin. The Signing Policy mechanism (Section 13) addresses this threat by making the Signer's verification posture explicit.

------

## 3. Terminology

- **Resource**: Any HTTP-delivered content (HTML, JS, images, JSON, etc.)
- **Signer**: The server or entity that holds a private key and signs Resources.
- **Verifier**: The client or tool that retrieves and verifies the Signature using the Signer’s public key.
- **Signature**: The result of signing a Resource body with a private key.
- **Public Key URI**: A URI where the Signer’s public key is published.
- **Signing Policy**: A declared assurance level indicating what the Signer verified about the content before signing. Expressed as a registered token in the `policy` parameter of the `Content-Signature` header (Section 13).

------

## 4. Quick Start

### 4.1 Example: P-256 (Default Suite)

**Step 1: Generate a Key Pair**

```
openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem  
openssl ec -in private-key.pem -pubout -out public-key.pem  
```

**Step 2: Sign Your Content**

```
openssl dgst -sha256 -sign private-key.pem -out signature.bin message.txt  
openssl base64 -in signature.bin -out signature.b64  
```

**Step 3: Serve Content with `Content-Signature` Header**

```
Content-Signature: alg=ecdsa-p256-sha256; key_uri=/.well-known/wcap-public-key.pem; sig=<base64 signature>
```

Ensure the public key is served at the path specified in `key_uri`.

### 4.2 Example: DD-ECIES (secp256k1 Suite)

This example demonstrates signing content using the `dd-ecies-secp256k1-sha256` algorithm suite. The signature uses ECDSA over secp256k1 with SHA-256, producing a 64-byte compact signature as defined in the DD-ECIES specification.

**Step 1: Generate a secp256k1 Key Pair**

```
# Generate a secp256k1 private key
openssl ecparam -name secp256k1 -genkey -noout -out private-key-secp256k1.pem

# Extract the compressed public key in PEM format
openssl ec -in private-key-secp256k1.pem -pubout -conv_form compressed -out public-key-secp256k1.pem
```

The public key MUST be a 33-byte compressed secp256k1 key (0x02 or 0x03 prefix byte) encoded in PEM format.

**Step 2: Sign Your Content**

Compute the SHA-256 hash of the content, then sign the hash using ECDSA with deterministic nonce generation (RFC 6979). The resulting signature MUST be a 64-byte compact format: `r (32 bytes) || s (32 bytes)`, with no DER encoding.

```
# Compute SHA-256 hash and sign with ECDSA-secp256k1
# Note: openssl produces DER-encoded signatures by default.
# A DD-ECIES-compatible tool or library must be used to produce
# the 64-byte compact (r || s) signature format.
openssl dgst -sha256 -sign private-key-secp256k1.pem -out signature-der.bin message.txt

# Convert DER signature to 64-byte compact (r || s) format and base64-encode.
# This step requires a tool that extracts the raw r and s integers from the
# DER encoding and concatenates them as 32-byte big-endian values.
# Example using a DD-ECIES library:
#   dd-ecies sign --key private-key-secp256k1.pem --input message.txt --output signature.b64
```

**Step 3: Serve Content with `Content-Signature` Header**

```
Content-Signature: alg=dd-ecies-secp256k1-sha256; key_uri=/.well-known/wcap-public-key-secp256k1.pem; sig=<base64 compact signature>
```

Optionally include a `kid` parameter to identify the key when the signer publishes multiple keys:

```
Content-Signature: alg=dd-ecies-secp256k1-sha256; key_uri=/.well-known/wcap-public-key-secp256k1.pem; sig=<base64 compact signature>; kid=secp256k1-2025
```

Ensure the compressed secp256k1 public key is served at the path specified in `key_uri`. See the Algorithm Suite Registry (Section 12) for full suite details and the DD-ECIES specification for the compact signature format.

------

## 5. Protocol Flow

1. The Verifier requests a Resource.
2. The Signer generates the Resource body and its Signature.
3. The Signer returns the Resource with the `Content-Signature` HTTP header.
4. The Verifier retrieves the public key from `key_uri`.
5. The Verifier checks the Signature against the Resource body.

If verification succeeds, the Resource is authentic. If it fails, the Resource is potentially tampered with.

------

## 6. Server (Signer) Requirements

### 6.1 Key Management

- MUST generate and securely store a public/private key pair.
- RECOMMENDED algorithm: ECDSA (P-256 with SHA-256).
- MAY support other modern, secure digital signature algorithms.
- SHOULD rotate keys periodically and expose metadata for revocation and validity.

### 6.2 Public Key Publication

- MUST publish public key at a stable `key_uri`, such as:

  ```
  /.well-known/wcap-public-key.pem
  ```

- MUST serve it over HTTPS with a valid certificate.

- Public key MUST be in PEM format.

- MAY include a revocation URI and expiration metadata inside the PEM file.

### 6.3 The `Content-Signature` Header

Header format:

```
Content-Signature: alg=<algorithm>; key_uri=<uri>; sig=<base64 signature>[; kid=<key-id>][; policy=<policy-token>]
```

#### 6.3.1 Parameters

- `alg`: REQUIRED. Algorithm suite identifier. MUST be a registered Algorithm Suite identifier from the Algorithm Suite Registry (Section 12). Valid values include:
  - `ecdsa-p256-sha256` — ECDSA with P-256 and SHA-256 (default suite).
  - `dd-ecies-secp256k1-sha256` — ECDSA with secp256k1 and SHA-256, as defined in the DD-ECIES specification.
- `key_uri`: REQUIRED. Public key location (relative or absolute URI). The key served at this URI MUST be in PEM format and MUST conform to the key format specified by the algorithm suite.
- `sig`: REQUIRED. Base64-encoded digital signature of the *exact response body*. The signature encoding and size MUST conform to the signature format specified by the algorithm suite.
- `kid`: OPTIONAL. Key ID string identifying which key the signer used. This parameter supports signers that publish multiple keys (e.g., for different algorithm suites or key rotation). When present, verifiers SHOULD use the `kid` value to select the appropriate public key for verification.
- `policy`: OPTIONAL. Signing policy token declaring what the Signer verified about the content before signing. MUST be a registered token from the Signing Policy Registry (Section 13). When absent, verifiers MUST treat the signature as making no claim beyond transport integrity. See Section 13 for registered policy tokens and their semantics.

#### 6.3.2 Algorithm-Specific Requirements

##### Default Suite (`ecdsa-p256-sha256`)

When `alg=ecdsa-p256-sha256`:

- The `sig` field MUST contain a base64-encoded DER-encoded ECDSA signature.
- The public key served at `key_uri` MUST be an uncompressed P-256 public key in PEM format.

##### DD-ECIES Suite (`dd-ecies-secp256k1-sha256`)

When `alg=dd-ecies-secp256k1-sha256`:

- The `sig` field MUST contain a base64-encoded 64-byte compact ECDSA signature in the format `r(32 bytes) || s(32 bytes)`, as defined in the DD-ECIES specification (Section: Signature Scheme).
- The public key served at `key_uri` MUST be a 33-byte compressed secp256k1 public key (0x02 or 0x03 prefix byte) encoded in PEM format.

------

## 7. Client (Verifier) Requirements

### 7.1 Signature Discovery

- MUST check for a `Content-Signature` header in HTTP responses.

### 7.2 Public Key Retrieval and Caching

- MUST fetch public key from `key_uri` (HTTPS RECOMMENDED).
- SHOULD cache keys securely for performance.
- MAY allow preloading or pinning of trusted keys.

### 7.3 Verification Logic

The Verifier MUST:

1. Parse the `Content-Signature` header.
2. Retrieve and cache the public key.
3. Verify the decoded `sig` against the exact body of the response.

### 7.4 Handling Verification Failure

- MUST NOT render or use the resource.
- SHOULD display a clear integrity warning or error.

------

## 8. Considerations for Specific Resource Types

### 8.1 Static Assets (JavaScript, CSS)

- SHOULD continue to use Subresource Integrity (SRI) when possible.
- WCAP MAY be used as a fallback or enhancement to SRI.

### 8.2 WebSockets

To sign WebSocket messages:

```
{
  "payload": { ... },
  "signature": "<base64 signature of payload>"
}
```

- Every message MUST be signed individually.
- Receiver MUST verify before acting on the message.

------

## 9. Security Considerations

- **Key Compromise**: MUST allow for key rotation and revocation.
- **Replay Attacks**: Resources MAY include timestamps or nonces.
- **Initial Trust**: Relies on HTTPS or DNSSEC for the first key fetch.
- **Signature Stripping**: Client SHOULD treat unsigned content as untrusted when a signature is expected.
- **Header Binding (Optional)**: Future versions MAY support binding signature to additional headers (e.g., MIME types, CSP).

------

## 10. Privacy Considerations

- Public key fetches can expose access patterns. To reduce this:
  - Verifiers SHOULD cache keys aggressively.
  - Verifiers MAY pre-fetch public keys for known hosts.
  - Alternative key distribution (e.g., IPFS, blockchain, DNSSEC) MAY enhance privacy and resilience.

------

## 11. Future Extensions (Non-normative)

- Verifiable streaming content using Merkle trees.
- Multi-signature schemes or endorsement chains.
- Web browser integration (e.g., extensions or native WCAP support).
- Blockchain-based or decentralized key registries.
- Cryptographic proof of signing policy compliance (e.g., zero-knowledge proofs that decryption succeeded without revealing key material).

------

## 12. Algorithm Suite Registry

### 12.1 Overview

WCAP supports pluggable algorithm suites, analogous to TLS cipher suites. An algorithm suite is a named combination of cryptographic algorithms — including an elliptic curve, signature algorithm, hash function, key format, and signature format — that signers and verifiers use to produce and verify content signatures.

The `alg` parameter in the `Content-Signature` header is the sole mechanism for algorithm negotiation. The core protocol logic MUST NOT contain algorithm-specific behavior; all algorithm-dependent details are encapsulated within the suite definition. This design ensures that new algorithm suites MAY be registered by defining the required fields in this registry without modifying the core protocol sections (Sections 5–7).

### 12.2 Registry Structure

Each algorithm suite entry in the registry MUST include the following fields:

| Field | Type | Description |
|-------|------|-------------|
| **Identifier** | string | The token used as the `alg` parameter value in the `Content-Signature` header. |
| **Curve** | string | The elliptic curve used for key generation and cryptographic operations. |
| **Signature Algorithm** | string | The digital signature algorithm. |
| **Hash Algorithm** | string | The hash function used for message digests. |
| **Key Format** | string | The encoding and size of the public key served at `key_uri`. |
| **Signature Format** | string | The encoding and size of the signature in the `sig` parameter. |
| **Reference** | string | The defining specification or standard for this suite. |

### 12.3 Registered Algorithm Suites

#### 12.3.1 `ecdsa-p256-sha256` (Default)

| Field | Value |
|-------|-------|
| **Identifier** | `ecdsa-p256-sha256` |
| **Curve** | P-256 (secp256r1), as defined in FIPS 186-4 |
| **Signature Algorithm** | ECDSA |
| **Hash Algorithm** | SHA-256 |
| **Key Format** | PEM-encoded uncompressed P-256 public key |
| **Signature Format** | DER-encoded ECDSA signature, base64 |
| **Reference** | FIPS 186-4, RFC 6979 |

This is the default algorithm suite. All conforming verifiers MUST support this suite.

#### 12.3.2 `dd-ecies-secp256k1-sha256`

| Field | Value |
|-------|-------|
| **Identifier** | `dd-ecies-secp256k1-sha256` |
| **Curve** | secp256k1, as defined in SEC 2 section 2.4.1 |
| **Signature Algorithm** | ECDSA with deterministic nonce generation per RFC 6979 |
| **Hash Algorithm** | SHA-256 |
| **Key Format** | PEM-encoded 33-byte compressed secp256k1 public key (0x02/0x03 prefix) |
| **Signature Format** | 64-byte compact signature (r \|\| s, 32 bytes each), base64-encoded |
| **Reference** | DD-ECIES: Digital Defiance Elliptic Curve Integrated Encryption Scheme, Version 1.0 |

Conforming verifiers SHOULD support this suite.

### 12.4 Algorithm Suite Negotiation

The `alg` parameter in the `Content-Signature` header (Section 6.3) is the sole negotiation mechanism for algorithm suites. A signer selects an algorithm suite and sets the `alg` parameter to the corresponding identifier. The verifier uses the `alg` value to determine which cryptographic operations to apply during signature verification.

The core protocol (Sections 5–7) MUST NOT contain algorithm-specific behavior. All algorithm-dependent logic — including key parsing, hash computation, and signature verification — MUST be delegated to the suite definition.

### 12.5 Handling Unrecognized Algorithm Suites

When a verifier encounters an `alg` value in the `Content-Signature` header that does not match any registered algorithm suite identifier, the verifier:

1. MUST reject the signature.
2. SHOULD report the unsupported algorithm in a diagnostic message or log entry, including the unrecognized `alg` value.
3. MUST NOT attempt to verify the signature using a different algorithm suite.

### 12.6 Extensibility

New algorithm suites MAY be registered by adding an entry to this registry with all required fields (Section 12.2). Registration of a new suite MUST NOT require modifications to the core protocol sections (Sections 5–7).

### 12.7 Verifier Support Requirements

- Verifiers MUST support the `ecdsa-p256-sha256` suite.
- Verifiers SHOULD support the `dd-ecies-secp256k1-sha256` suite.
- Verifiers MAY support additional registered suites.

------

## 13. Signing Policy Declarations

### 13.1 Overview

A WCAP signature cryptographically proves that the Signer held the private key corresponding to the published public key and that the signed bytes were not altered after signing. It does not, by itself, say anything about what the Signer verified about the content *before* signing.

For many deployments this distinction matters. A Signer that retrieves content from encrypted block storage, a third-party origin, or a peer-to-peer network may have verified the content's integrity and provenance before serving it — or it may have forwarded bytes without any such check. A verifier that cares about provenance, not just transport integrity, needs a way to distinguish these cases.

The `policy` parameter in the `Content-Signature` header (Section 6.3.1) allows a Signer to declare, as part of the signed response, what it verified before signing. The declaration is a registered token from the Signing Policy Registry defined in this section.

**Important limitation:** A signing policy declaration is a *self-asserted claim* by the Signer. It is not cryptographically provable by the Verifier. Verifiers that rely on policy declarations MUST establish trust in the Signer's honesty through out-of-band means (e.g., audits, published signing policy documentation, or reputation). The `policy` parameter is a transparency mechanism, not a proof mechanism.

### 13.2 Registry Structure

Each entry in the Signing Policy Registry MUST include the following fields:

| Field | Type | Description |
|-------|------|-------------|
| **Token** | string | The value used as the `policy` parameter in the `Content-Signature` header. MUST consist only of lowercase ASCII letters, digits, and hyphens. |
| **Name** | string | A short human-readable name for the policy. |
| **Assurance Level** | integer | A numeric ordering from 0 (no claim) to 3 (strongest claim), allowing verifiers to enforce a minimum assurance level. |
| **Description** | string | A normative description of what the Signer MUST have verified before signing under this policy. |
| **Example Use Case** | string | A non-normative example of a system that would use this policy. |

### 13.3 Registered Signing Policies

#### 13.3.1 `none` (Assurance Level 0)

| Field | Value |
|-------|-------|
| **Token** | `none` |
| **Name** | No Policy Claim |
| **Assurance Level** | 0 |
| **Description** | The Signer makes no claim about what was verified before signing. The signature attests only to transport integrity: the bytes were not modified after the Signer produced the response. This is equivalent to omitting the `policy` parameter entirely. |
| **Example Use Case** | A reverse proxy that signs all outbound responses without inspecting content. |

#### 13.3.2 `content-verified` (Assurance Level 1)

| Field | Value |
|-------|-------|
| **Token** | `content-verified` |
| **Name** | Content Verified |
| **Assurance Level** | 1 |
| **Description** | The Signer verified that the content was retrievable and well-formed before signing. The Signer MUST have successfully fetched or reconstructed the content from its authoritative source (e.g., a database, object store, or block storage system) and confirmed that the retrieval did not produce an error. The Signer makes no claim about the content's origin or ownership. |
| **Example Use Case** | A file server that reads from a local disk and confirms the file exists and is readable before signing the response. |

#### 13.3.3 `origin-verified` (Assurance Level 2)

| Field | Value |
|-------|-------|
| **Token** | `origin-verified` |
| **Name** | Origin Verified |
| **Assurance Level** | 2 |
| **Description** | The Signer verified that the content was retrieved from a known, trusted origin and that the retrieval was authenticated. The Signer MUST have confirmed the identity of the upstream source (e.g., via a verified signature, authenticated storage credential, or cryptographic commitment) before signing. The Signer makes no claim about the content's semantic validity or ownership by a specific end user. |
| **Example Use Case** | A CDN edge node that fetches content from an origin server over a mutually authenticated TLS connection and verifies the origin's identity certificate before signing the response. |

#### 13.3.4 `decryption-verified` (Assurance Level 3)

| Field | Value |
|-------|-------|
| **Token** | `decryption-verified` |
| **Name** | Decryption Verified |
| **Assurance Level** | 3 |
| **Description** | The Signer retrieved the content from encrypted storage, successfully decrypted it using the appropriate key material, and is signing the plaintext result. Successful decryption constitutes cryptographic proof of content integrity and authenticity: a ciphertext that decrypts without error under the correct key was produced by an entity that held that key and has not been tampered with since encryption. The Signer MUST NOT set this policy if decryption produced an error or if the content was served from a cache without re-verifying the ciphertext. |
| **Example Use Case** | A file-serving system that stores files as ECIES-encrypted blocks. Before signing the response, the server decrypts the block using the member's key material. A successful decryption guarantees the block was encrypted by a legitimate participant and has not been corrupted or substituted. |

### 13.4 Signer Requirements

When a Signer includes a `policy` parameter in the `Content-Signature` header:

1. The Signer MUST have performed all verifications described in the policy's normative description before signing.
2. The Signer MUST NOT assert a policy token for verifications it did not perform.
3. The Signer SHOULD document its signing policy publicly (e.g., in a `/.well-known/wcap-signing-policy.json` resource) so that verifiers can audit the claim.
4. If the Signer cannot complete the required verifications for a given policy (e.g., decryption fails), the Signer MUST either omit the `policy` parameter, use a lower-assurance policy token, or omit the `Content-Signature` header entirely.

### 13.5 Verifier Requirements

When a Verifier processes a `Content-Signature` header containing a `policy` parameter:

1. The Verifier MUST first verify the cryptographic signature as defined in Section 7.3. A valid signature is a prerequisite for trusting any policy claim.
2. The Verifier MAY enforce a minimum assurance level. If the declared policy's assurance level is below the Verifier's required minimum, the Verifier SHOULD treat the response as if no policy was declared.
3. The Verifier MUST NOT treat a policy declaration as cryptographic proof of the claimed verifications. Policy compliance is a declaration of intent and practice, not a provable property.
4. When a `policy` parameter contains an unrecognized token, the Verifier SHOULD treat it as assurance level 0 (`none`) and MAY log a diagnostic warning.

### 13.6 Example Header with Policy

```
Content-Signature: alg=dd-ecies-secp256k1-sha256; key_uri=/.well-known/wcap-public-key-secp256k1.pem; sig=<base64 compact signature>; policy=decryption-verified
```

This header declares that the Signer used the `dd-ecies-secp256k1-sha256` algorithm suite, the content was retrieved from encrypted storage and successfully decrypted before signing, and the signature covers the exact plaintext bytes in the response body.

### 13.7 Extensibility

New signing policy tokens MAY be registered by adding an entry to this registry with all required fields (Section 13.2). Registration of a new policy MUST NOT require modifications to the core protocol sections (Sections 5–7) or the Algorithm Suite Registry (Section 12). Policy tokens are independent of algorithm suites; any policy MAY be combined with any algorithm suite.

------

## License

This specification is published under the MIT License.
 (C) 2024–2026 Digital Defiance. Contributions welcome via GitHub.
