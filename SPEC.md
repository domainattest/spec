# DomainAttest

## An open protocol for instant domain ownership verification

## Version 1.0

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as described in RFC 2119.

---

## 1. Purpose and scope

DomainAttest defines how a domain registrar issues a signed, short-lived attestation that an authenticated account controls a specific domain, and how a relying party verifies that attestation.

In scope for v1: ownership attestation for a single domain, initiated by the domain holder, delivered to a named relying party.

Out of scope for v1: bulk attestation, ownership transfer, registrant identity disclosure, DNS configuration. See Section 11 for the v2 candidate list.

## 2. Roles

- **Holder.** The person or entity with an authenticated account controlling the domain at the registrar.
- **Registrar.** The sponsoring registrar of record for the domain. Issues attestations.
- **Relying Party (RP).** The service requesting proof of control, such as a marketplace. Consumes attestations.
- **Hub (optional).** An aggregation service that routes verification requests to registrars and returns attestations to RPs through a single API. The hub transports attestations; it MUST NOT sign or modify them.

## 3. Protocol flow (direct mode)

DomainAttest is a profile of the OAuth 2.0 authorization code flow with PKCE (RFC 6749, RFC 7636). It introduces no new cryptographic primitives.

### 3.1 Discovery

The RP determines the domain's sponsoring registrar via RDAP lookup, then fetches the registrar's DomainAttest metadata from:

```
https://{registrar-attest-host}/.well-known/domainattest-configuration
```

Metadata MUST include: `issuer`, `authorization_endpoint`, `token_endpoint`, `jwks_uri`, `attest_versions_supported`, and `modes_supported` (any of `direct`, `hub`). See `examples/domainattest-configuration.json`.

### 3.2 Authorization request

The RP redirects the holder's user agent to the registrar's authorization endpoint with:

| Parameter | Requirement |
|---|---|
| `response_type` | MUST be `code` |
| `client_id` | The RP's registered identifier |
| `redirect_uri` | MUST exactly match a registered URI |
| `domain` | The domain to verify, lowercase, punycode for IDNs |
| `state` | RECOMMENDED, per OAuth best practice |
| `code_challenge`, `code_challenge_method` | REQUIRED, `S256` |

### 3.3 Authentication and consent

The registrar authenticates the holder using its normal account login, including any second factor enabled on the account. The registrar MUST verify that the authenticated account controls `domain`.

The consent screen MUST name the RP and the domain explicitly, in the form: "Verify to {RP name} that this account controls {domain}."

### 3.4 Attestation issuance

On approval, the registrar redirects to `redirect_uri` with an authorization code. The RP exchanges the code at the token endpoint, presenting the PKCE verifier, and receives the attestation in the response field `attestation`.

Authorization codes MUST be single-use and SHOULD expire within 60 seconds.

### 3.5 Validation

The RP MUST:

1. Verify the JWS signature against a key from the registrar's published JWKS, selected by `kid`.
2. Verify `iss` matches the discovered registrar.
3. Verify `sub` equals the requested domain.
4. Verify `aud` equals the RP's own `client_id`.
5. Verify current time is within `iat` and `exp`.
6. Verify `jti` has not been seen within the validity window.

Failure of any check MUST result in rejection of the attestation.

## 4. Attestation format

The attestation is a JWS-signed JWT.

### 4.1 Required claims

| Claim | Meaning |
|---|---|
| `iss` | Registrar identifier: the registrar's DomainAttest issuer URL. Registrars SHOULD also include their IANA registrar ID in the `attest_iana_id` claim |
| `sub` | The verified domain, lowercase, punycode for IDNs |
| `aud` | The RP's `client_id`. The attestation is valid only for this RP |
| `iat` | Issued-at timestamp |
| `exp` | Expiry. MUST be no more than 15 minutes after `iat` |
| `jti` | Unique attestation ID |
| `attest_ver` | Protocol version. `"1"` for this specification |

### 4.2 Optional claims

| Claim | Meaning |
|---|---|
| `acct_hash` | Salted hash of the registrar account identifier. Lets an RP detect that multiple domains were verified by the same account without learning the account. The salt MUST be stable per registrar-RP pair and MUST NOT be reversible |
| `locked` | Boolean. Whether a transfer lock is active on the domain |
| `attest_iana_id` | The registrar's IANA ID |

### 4.3 Excluded data

Attestations MUST NOT include registrant name, organization, email, address, phone, or any other Whois or account contact data. The attestation proves control, not identity.

### 4.4 Signature

`ES256` is REQUIRED to support; `RS256` MAY be supported. Keys are published at the `jwks_uri` declared in registrar metadata. The JWS header MUST include `kid`. Registrars SHOULD rotate keys periodically and MUST retain retired public keys in the JWKS for at least the maximum attestation lifetime after rotation.

## 5. Hub mode

The hub exposes the same authorization flow to RPs and proxies discovery and redirection to the correct registrar.

- The RP integrates one API and one `client_id` registration instead of one per registrar.
- The hub performs registrar discovery and metadata caching.
- The attestation returned to the RP is the registrar-signed JWT, unmodified. The RP validates it exactly as in direct mode (Section 3.5). The hub adds no signature and cannot forge attestations.
- Registrars declare in metadata whether they accept authorization traffic directly, via hub, or both.

Because the attestation format and validation are identical in both modes, direct and hub participants interoperate without coordination.

## 6. Errors

Standard OAuth error responses apply, extended with:

| Code | Meaning |
|---|---|
| `attest_domain_not_held` | The authenticated account does not control the requested domain |
| `attest_domain_ineligible` | The domain exists at the registrar but is in a state that prevents attestation, such as a dispute, expiry hold, or court order |
| `attest_unsupported_tld` | The registrar does not offer DomainAttest for this TLD |
| `attest_rate_limited` | Too many attestation requests for this domain in a rolling window |

Error responses MUST NOT reveal to unauthenticated parties whether a domain or account exists at the registrar.

## 7. Security considerations

**No bearer secrets.** There is no static verification key. Nothing the holder can copy, paste, or be phished out of confers verification power elsewhere.

**Audience binding.** `aud` scopes each attestation to one RP. An attestation captured in transit or leaked from an RP database is useless anywhere else.

**Short expiry.** The 15-minute maximum lifetime bounds the window between verification and the RP's decision. RPs needing ongoing assurance re-verify. v1 deliberately defines no long-lived tokens.

**Replay.** RPs MUST track `jti` values within the validity window and reject duplicates.

**Phishing surface.** The consent screen lives on the registrar's own domain under its existing login. The protocol introduces no new location where a holder enters credentials. RPs MUST initiate flows only via the discovered authorization endpoint and MUST NOT embed or frame registrar login.

**Registrar key compromise.** A compromised signing key allows forged attestations for that registrar's domains only, mitigated by rotation and short lifetimes. This is strictly better than the status quo, where a compromised DNS panel passes TXT verification silently and indefinitely.

**Privacy.** Attestations carry no registrant data. The registrar learns which RP the holder is verifying to, which the holder explicitly consents to on the consent screen.

## 8. RP registration

v1 uses static registration: an RP registers its `client_id` and redirect URIs with each registrar it integrates directly, or once with a hub. Dynamic client registration is deferred to v2.

## 9. Registrar implementation checklist

1. Serve `/.well-known/domainattest-configuration`.
2. Implement the authorization endpoint over existing account login: authenticate, check domain control, render consent, issue code.
3. Implement the token endpoint: exchange code plus PKCE verifier for a signed attestation.
4. Publish JWKS and manage key rotation.
5. Register accepted RPs, or delegate RP management to a hub.

## 10. Versioning and governance

This specification is published under CC BY 4.0 in a public repository. Changes proceed by versioned releases. v1 is stable; implementations targeting v1 will not be broken by future versions. Implementation reports and issues are accepted in the repository.

## 11. Candidate v2 features

- Bulk attestation for portfolio verification in one consent.
- Standing verification: revocable, longer-lived attestations for marketplace-of-record relationships.
- An optional settlement path for sales where buyer and seller hold accounts at the same registrar.
- Dynamic RP registration.
- Agent-initiated flows with delegated holder consent.
