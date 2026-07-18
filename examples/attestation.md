# Example attestation

A decoded DomainAttest v1 attestation. On the wire this is a compact JWS.

## JWS header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "2026-07-a"
}
```

## Claims

```json
{
  "iss": "https://attest.example-registrar.com",
  "attest_iana_id": 1234,
  "sub": "exampledomain.com",
  "aud": "marketplace-client-8842",
  "iat": 1784224800,
  "exp": 1784225700,
  "jti": "0198f2a4-7c1e-7b3a-9f4d-2e6b8c1a5d90",
  "attest_ver": "1",
  "acct_hash": "u5Zt3...redacted...9qLm",
  "locked": true
}
```

Reading this attestation: the registrar at `attest.example-registrar.com` (IANA ID 1234) attests that an authenticated account controls `exampledomain.com`. The attestation was issued for the relying party `marketplace-client-8842` only, is valid for 15 minutes, and the domain currently has a transfer lock. The `acct_hash` lets this RP recognize other domains verified by the same account without learning the account identity.

## Example error response

Authorization endpoint response when the authenticated account does not control the requested domain:

```json
{
  "error": "attest_domain_not_held",
  "error_description": "The authenticated account does not control the requested domain."
}
```
