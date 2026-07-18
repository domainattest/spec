# DomainAttest

**Instant, cryptographically verifiable proof of domain ownership.**

DomainAttest lets a domain registrar issue a signed, short-lived attestation that an authenticated account controls a specific domain, and lets any relying party (typically a domain marketplace) verify that attestation in one second.

No TXT records. No name server changes. No propagation. No stale verifications.

## The problem

Proving you own a domain today means editing DNS and waiting for propagation. Because that is slow and manual, the industry works around it in two ways that are worse than the friction itself:

- **Some marketplaces skip verification entirely**, enabling front-running and fraudulent listings of domains the lister does not own.
- **Most others verify once and never again**, so verification goes stale when a domain changes hands, and the rightful new owner has to fight to displace it.

The registrar already knows who controls the domain. DomainAttest is the missing piece: a standard way for the registrar to say so.

## How it works

An OAuth 2.0 authorization code flow with PKCE. The domain holder clicks "Verify with your registrar," logs in with their existing account and 2FA, and approves. The registrar returns a signed JWT attesting that the authenticated account controls the domain, scoped to the requesting relying party, expiring within 15 minutes.

```
Holder ──> Marketplace ──redirect──> Registrar login + consent
                                          │
Marketplace <──signed attestation (JWT)───┘
```

The relying party validates the signature against the registrar's published keys. Done.

No new cryptography. No new identity system. No bearer secrets to phish.

## Read the spec

- **[SPEC.md](SPEC.md)**: the v1 protocol specification
- **[docs/WHITEPAPER.md](docs/WHITEPAPER.md)**: motivation and benefits for sellers, registrars, marketplaces, and buyers
- **[examples/](examples/)**: registrar metadata, decoded attestations, error responses

## Implementing

**Registrars** expose one authorization endpoint and one token endpoint over their existing login, and publish a JWKS. Typical effort with a modern auth stack: two to four engineering weeks. See the [implementation checklist](SPEC.md#9-registrar-implementation-checklist).

**Relying parties** either integrate registrars directly per the spec, or integrate the hosted DomainAttest Hub once and gain coverage of all connected registrars. Attestations are registrar-signed and independently verifiable in both modes.

Known implementations are listed in [IMPLEMENTATIONS.md](IMPLEMENTATIONS.md).

## Design principles

- **Open.** Public specification, free to implement, no permission required.
- **Minimal.** One flow, one attestation format, one signature scheme.
- **Built on existing standards.** OAuth 2.0, PKCE, JWT/JWS, JWKS.
- **No bearer secrets.** Nothing a holder can copy, leak, or be phished out of.
- **Time-bound by design.** Ownership is a fact about now. Attestations expire in minutes and re-verification costs one click, so verification stays current instead of going stale.
- **Privacy-preserving.** Attestations prove control, not identity. No registrant data is exposed.

## Status

v1 is stable. Implementations targeting v1 will not be broken by future versions. See [CONTRIBUTING.md](CONTRIBUTING.md) for how changes are handled and how to report implementation feedback.

## License

The specification and documentation in this repository are licensed under CC BY 4.0.

---

DomainAttest was created by [Atom](https://www.atom.com), which operates the reference implementation and the DomainAttest Hub. Protocol home: [domainattest.org](https://domainattest.org).
