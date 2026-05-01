# Contributing to WCAP

Thank you for your interest in the Web Content Authenticity Protocol. This document explains how to contribute to the specification and its ecosystem.

## Ways to Contribute

### Report Issues

If you find an ambiguity, error, or gap in the specification, [open an issue](https://github.com/Digital-Defiance/WCAP/issues) with:

- The section number and text in question
- What you expected vs. what the spec says
- A suggested fix, if you have one

### Propose Specification Changes

1. Fork the repository.
2. Edit the canonical Markdown file: `Web Content Authenticity Protocol (WCAP).md`.
3. Regenerate the other formats (see below).
4. Open a pull request with a clear description of the change and which sections are affected.

Changes to normative text (anything using MUST, SHALL, SHOULD, MAY) require careful review. Non-normative changes (examples, clarifications, typo fixes) are welcome and typically merged quickly.

### Register a New Algorithm Suite

To propose a new algorithm suite for the Algorithm Suite Registry (Section 12):

1. Open an issue titled "Algorithm Suite Registration: `<your-suite-identifier>`".
2. Include all required fields:
   - **Identifier**: The `alg` parameter token (e.g., `ecdsa-p384-sha384`)
   - **Curve**: The elliptic curve
   - **Signature Algorithm**: The signature scheme
   - **Hash Algorithm**: The hash function
   - **Key Format**: Public key encoding and size
   - **Signature Format**: Signature encoding and size
   - **Reference**: The defining specification (must be publicly available)
3. Include at least one test vector (key pair, message, signature) so implementers can validate compatibility.
4. If accepted, submit a PR adding the suite to Section 12.3 of the specification.

New suites MUST NOT require changes to the core protocol (Sections 1–7). If your suite needs protocol changes, that's a larger discussion — open an issue first.

### Add an Implementation

If you've built a WCAP signer or verifier, add it to the implementations table in `README.md`:

1. Fork the repository.
2. Add a row to the Implementations table with your project name, language, and link.
3. Open a PR.

### Build a Browser Extension or Verifier

We're actively looking for contributors to build:

- A browser extension that checks `Content-Signature` headers and shows a trust indicator
- A CLI verification tool
- Client-side JavaScript/TypeScript verification libraries

If you're interested, open an issue to discuss the approach before starting.

## Regenerating Formats

After editing the canonical Markdown, regenerate the other formats:

```bash
# Prerequisites: pandoc, xml2rfc
# Install: brew install pandoc && pip install xml2rfc

# HTML (standalone)
pandoc "Web Content Authenticity Protocol (WCAP).md" \
    -o "Web Content Authenticity Protocol (WCAP).html" \
    --standalone --metadata title="WCAP"

# HTML (no style / fragment)
pandoc "Web Content Authenticity Protocol (WCAP).md" \
    -o "Web Content Authenticity Protocol (WCAP)-ns.html" \
    --metadata title="WCAP"

# Plain text
pandoc "Web Content Authenticity Protocol (WCAP).md" \
    -o "WebContentAuthenticityProtocol.txt" \
    --to plain --wrap=auto --columns=80

# IETF Internet-Draft (from XML source — edit XML separately)
xml2rfc --v3 draft-digitaldefiance-wcap-00.xml --text
xml2rfc --v3 draft-digitaldefiance-wcap-00.xml --html
```

The IETF XML draft (`draft-digitaldefiance-wcap-00.xml`) is maintained separately from the Markdown because it uses RFC XML v3 structure. When the Markdown changes, update the XML to match.

## Code of Conduct

Be respectful and constructive. This is a technical specification — focus on the protocol, not the people.

## License

By contributing, you agree that your contributions will be licensed under the same [MIT License](LICENSE) as the specification.
