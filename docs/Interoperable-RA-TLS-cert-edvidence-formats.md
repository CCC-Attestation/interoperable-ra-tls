Interoperable RA-TLS X.509 Cert and Evidence Formats
====

# X.509 Cert with Evidence and Endorsements Extensions

A X.509 cert for RA-TLS can either be self-signed or CA-signed. It must contain an extension for evidence, and can (optionally) contain an extension for endorsements. The evidence and endorsement extensions are defined in the latest [TCG DICE specification draft](https://members.trustedcomputinggroup.org/wg/DICE/document/36858).

- Evidence (required): X.509 cert extension, a byte string of encoded tagged CBOR data
    - OID: `tcg-dice-conceptual-message-wrapper` (2.23.133.5.4.9)
    - CBOR tag: a CBOR tag registered with IANA or some other registry serves as evidence format ID
    - CBOR data: this data covered by the tag contains the evidence (including custom claims). Its format is defined by the CBOR tag.
    - Note: the DICE specification states that the criticality flag of this extension SHOULD be marked critical, but popular TLS libraries (such as openssl and mbedtls) return errors in verification of an X.509 cert with this extension if its criticality flag is set. Until the issue of library support is resolved, the criticality flag of this extension should be cleared.
- Endorsement (optional): X.509 cert extension, also a byte string of encoded tagged CBOR data. When this extension is present, its value can be used as endorsements for verification of the evidence in the evidence extension.
    - OID: `tcg-dice-endorsement-manifest` (2.23.133.5.4.2)
    - CBOR tag: the same tag as that for evidence extension
    - CBOR data: this data covered by the tag contains the endorsements for verification of the evidence in the same cert. Its format is defined by the CBOR tag.

# Evidence Custom Claims
The evidence extension must include a `pubkey-hash` claim, and can optionally include a `nonce` claim. All other custom claims will be ignored.

- `pubkey-hash` (required): holds `pubkey-hash-value`, a byte string of the definite-length encoded CBOR array `hash-entry` defined in [CoSWID](https://github.com/sacmwg/draft-ietf-sacm-coswid/blob/master/draft-ietf-sacm-coswid.md) and used by [CoRIM](https://github.com/ietf-rats/ietf-corim-cddl/blob/main/concise-mid-tag.cddl).
    - `hash-entry` is a CBOR array with two entries: `[ hash-alg-id, hash-value]`, where:
        - `hash-alg-id` is an unsigned integer identifying the hash algorithm, as registered with [IANA](https://www.iana.org/assignments/named-information/named-information.xhtml).
            -  For interoperable RA-TLS, the verifier of `hash-entry' must support hash algorithms sha-256 (ID 1), sha-384 (ID 7) and sha-512 (ID 8).
        - `hash-value` is a byte string holding the hash value of the X.509 cert `SubjectPublicKeyInfo` object value in DER encoding.
            - Note: the `SubjectPublicKeyInfo` object includes both the `algorithm` and `subjectPublicKey` fields.
- `nonce` (optional): holds `nonce-value`, a nonce as a byte string. There is no restriction as to how many bytes the nonce value should be.
    - This claim is not present if the X.509 cert does not support pre-session freshness.
