# Contributing

## Specification status

DomainAttest v1 is stable and final. Changes to v1 are limited to errata: corrections that fix errors without changing protocol behavior. New capabilities land in future versions.

This model exists so implementers can build against v1 with confidence that it will not move under them.

## What we want from you

**Implementation reports.** If you have implemented DomainAttest as a registrar, relying party, or hub, open an issue titled `Implementation report: {name}` describing what you built, what was unclear in the spec, and anything that surprised you. Working implementations are the strongest input to v2.

**Errata.** If the spec contains an error, ambiguity, or contradiction, open an issue with the section number and the problem. Confirmed errata are fixed in-place with a changelog entry.

**v2 proposals.** Feature proposals are welcome as issues labeled `v2`. The current candidate list is in [SPEC.md Section 11](SPEC.md#11-candidate-v2-features). Proposals grounded in a shipped implementation carry the most weight.

**Security issues.** Do not open public issues for vulnerabilities in the protocol design. Report them privately to the security contact in the repository profile. Vulnerabilities in a specific implementation should go to that implementer.

## What we do not do here

We do not run design discussions for v1. The spec shipped; the way to disagree with it productively is to implement it, hit the problem for real, and file the report.

## Listing your implementation

Once live, add yourself to [IMPLEMENTATIONS.md](IMPLEMENTATIONS.md) via pull request: name, role (registrar, RP, hub), modes supported, and a link to your DomainAttest documentation or metadata endpoint.
