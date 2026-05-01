# Web Content Authenticity Protocol (WCAP)

**Version 0.3 — Draft**

WCAP is a protocol for ensuring the end-to-end authenticity and integrity of web resources. It allows a server (the **Signer**) to digitally sign HTTP responses so that a client (the **Verifier**) can cryptographically confirm the content hasn't been tampered with — even if a CDN, proxy, or hosting server is compromised.

WCAP is transport-agnostic. It works over HTTP, peer-to-peer networks, WebSockets, or any other delivery channel.

## How It Works

1. The Signer generates a key pair and publishes the public key at a well-known URI.
2. For each response, the Signer computes a digital signature over the response body.
3. The Signer includes the signature in a `Content-Signature` HTTP header.
4. The Verifier fetches the public key and verifies the signature against the response body.

```
Content-Signature: alg=ecdsa-p256-sha256;
    key_uri=/.well-known/wcap-public-key.pem;
    sig=<base64 signature>
```

## Quick Start

Generate a key pair and sign content with OpenSSL:

```bash
# Generate P-256 key pair
openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem
openssl ec -in private-key.pem -pubout -out public-key.pem

# Sign content
openssl dgst -sha256 -sign private-key.pem -out signature.bin message.txt
openssl base64 -in signature.bin -out signature.b64
```

Then serve the response with the `Content-Signature` header and publish the public key at `/.well-known/wcap-public-key.pem`.

See the [full specification](Web%20Content%20Authenticity%20Protocol%20(WCAP).md) for complete details.

## Algorithm Suites

WCAP supports pluggable algorithm suites, similar to TLS cipher suites. Two suites are currently registered:

| Suite | Curve | Signature Format | Status |
|-------|-------|-----------------|--------|
| `ecdsa-p256-sha256` | P-256 | DER-encoded ECDSA, base64 | **Default** — MUST support |
| `dd-ecies-secp256k1-sha256` | secp256k1 | 64-byte compact (r‖s), base64 | SHOULD support |

New suites can be registered without modifying the core protocol. See Section 12 of the specification.

## Signing Policy Declarations

Version 0.3 adds an optional `policy` parameter to the `Content-Signature` header. A Signer can declare what it verified about the content *before* signing, allowing verifiers to distinguish a transport-integrity-only signature from one backed by deeper content verification.

| Token | Assurance | Meaning |
|-------|-----------|---------|
| *(absent)* | 0 | No claim beyond transport integrity |
| `none` | 0 | Explicit no-claim declaration |
| `content-verified` | 1 | Content was retrievable and well-formed |
| `origin-verified` | 2 | Upstream source identity was authenticated |
| `decryption-verified` | 3 | Content was decrypted from encrypted storage; successful decryption proves integrity |

Example:

```
Content-Signature: alg=dd-ecies-secp256k1-sha256;
    key_uri=/.well-known/wcap-public-key-secp256k1.pem;
    sig=<base64 compact signature>;
    policy=decryption-verified
```

Policy declarations are self-asserted — not cryptographically provable. Verifiers MUST establish trust through out-of-band means. See Section 13 of the specification.

## Specification Formats

The WCAP specification is available in multiple formats:

| Format | File |
|--------|------|
| Markdown (canonical) | [Web Content Authenticity Protocol (WCAP).md](Web%20Content%20Authenticity%20Protocol%20(WCAP).md) |
| IETF Internet-Draft XML (xml2rfc v3) | [draft-digitaldefiance-wcap-00.xml](draft-digitaldefiance-wcap-00.xml) |
| IETF Internet-Draft Text | [draft-digitaldefiance-wcap-00.txt](draft-digitaldefiance-wcap-00.txt) |
| IETF Internet-Draft HTML | [draft-digitaldefiance-wcap-00.html](draft-digitaldefiance-wcap-00.html) |
| HTML (standalone) | [Web Content Authenticity Protocol (WCAP).html](Web%20Content%20Authenticity%20Protocol%20(WCAP).html) |
| Plain text | [WebContentAuthenticityProtocol.txt](WebContentAuthenticityProtocol.txt) |

The Markdown file is the canonical source. Other formats are generated from it (or from the IETF XML for the draft outputs).

## Implementations

| Implementation | Language | Status |
|---------------|----------|--------|
| [Digital Burnbag](https://github.com/Digital-Defiance/BrightChain) | TypeScript (Express) | In development |

If you've implemented WCAP, please open a PR to add your implementation to this table.

## Related Specifications

- **[DD-ECIES Specification](https://github.com/Digital-Defiance/ecies-lib/blob/main/docs/DD-ECIES-SPECIFICATION.md)** — Defines the `dd-ecies-secp256k1-sha256` algorithm suite's cryptographic operations, wire formats, and test vectors.

## Threat Model

WCAP protects against:

- Content injection or modification in transit
- Compromised CDNs, proxies, or hosting infrastructure
- DNS manipulation and rerouting
- TLS interception via rogue or coerced CAs
- Government-forced censorship or content alteration
- Signer coercion (key compromise detection via rotation/revocation)
- Provenance ambiguity (Signer forwarding unverified upstream content without disclosure)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on proposing changes, registering new algorithm suites, and reporting issues.

## License

This specification is published under the [MIT License](LICENSE).

© 2024–2026 [Digital Defiance](https://digitaldefiance.org). Contributions welcome.
