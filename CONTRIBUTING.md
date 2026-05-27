# Contributing to ORP

Thanks for your interest. This spec exists because the open web needs a shared substrate the walled gardens already have internally, and that substrate only works if many vendors and operators participate in defining it. Your contribution — large or small, technical or strategic, code or commentary — is the point.

This document covers how to contribute, what kinds of contributions land easiest, and how proposals make their way from issue to accepted change.

---

## 1. The kinds of contributions that matter

### Wire format and schema

The OpenRTB extension structure, the JSON Schemas for embeddings and provenance manifests, the gRPC IDL. Changes here ripple through every implementation; the bar is correspondingly high. Open an issue first; expect RFC-style discussion.

### Latent Space Specifications (LSS)

The semantic definitions for each first-class embedding type. Each LSS is its own document (`lss/<embedding_type>/SPEC.md`). New LSSs and major LSS revisions go through RFC; minor LSS revisions (added axes, added invariants, added regime labels) can go through standard pull request.

This is where the most consequential contributions live. If you have production experience with an embedding type and a view on what makes it conformant, your input is high-leverage.

### Regime taxonomy

The closed, versioned taxonomy of regime labels (`regime-taxonomy/`). New labels are added through RFC. The bar: labels must be definitionally crisp, mutually exclusive within their namespace, and operationally useful to planners and analysts.

### Reference implementations

The reference SDKs (TypeScript, Python), the reference temporal encoder, the Prebid module, the ARTF sidecar reference. These are Apache 2.0; pull requests welcome with appropriate test coverage and documentation.

### Reference evaluation sets

Test sets per LSS that providers run their models against to attest conformance. Building and maintaining these is real work; contributors who do it are credited as data-set authors and get significant influence on what gates the LSS specifies.

### Documentation

The docs in `docs/`. Smaller changes — typo fixes, clarifications, examples — go through standard pull request. Larger restructurings should start with an issue.

### Strategic and editorial input

Not every contribution is code. Strategic input on positioning, governance posture, RFC priorities, working-group formation — all valuable. Open an issue and label it `strategy`.

---

## 2. Before you contribute

### Read these first

- [README.md](README.md) — project overview
- [SPEC.md](SPEC.md) — the formal v0.1 specification
- [docs/01-architecture.md](docs/01-architecture.md) — the four phases and L1/L2/L3 layer model
- [docs/06-governance.md](docs/06-governance.md) — the open / proprietary boundary and the RFC process

For LSS contributions specifically:

- [docs/02-latent-space.md](docs/02-latent-space.md) — the LSS framework
- The existing LSS documents in `lss/` if any are published for the embedding type you're working on

### Check whether your contribution already has an issue

The repository has open issues for the major outstanding work items. Adding to an existing thread is usually preferable to opening a new one.

### Understand the governance posture

Two principles matter most:

1. **ORP is a complementary forward prior, not a substitute.** Proposals that frame ORP as displacing OpenRTB, identity middleware, SSPs, DSPs, or measurement vendors will face structural resistance regardless of technical merit. The positioning is explicit ([docs/06-governance.md §1](docs/06-governance.md#1-the-forward-prior-positioning)).
2. **Interfaces are open; implementations are proprietary.** Proposals that would standardize what should be proprietary (specific models, specific rankers, specific commercial terms) will be rejected. Proposals that strengthen the open interface layer (LSS, wire format, conformance, interpretability) are welcome.

---

## 3. Contribution paths

### Path A — small change

Typos, clarifications, single-line corrections, additional examples. Standard pull request flow:

1. Fork the repository
2. Branch from `main`
3. Make the change
4. Run any applicable tests
5. Open a pull request with a brief description

Smaller changes typically merge within a few days.

### Path B — substantive but bounded change

New regime labels (within an existing namespace), minor LSS revisions (adding invariants or cross-space relationships), new reference SDK features, documentation expansions.

Process:
1. Open an issue describing what you want to change and why
2. Discuss with maintainers and other interested parties for at least 7 days
3. If consensus emerges, open a pull request implementing the change
4. Pull request reviewed against the discussion outcome

### Path C — RFC-required change

Wire format changes, major LSS revisions, new first-class embedding types, regime taxonomy structural changes, governance changes.

Process:
1. Open an issue tagged `rfc-needed`
2. Discuss for at least 14 days; gather affected-party input
3. Draft a formal RFC document in `rfcs/` covering:
   - Motivation
   - Proposed design
   - Alternatives considered
   - Migration path for existing implementations
   - Impact on backwards compatibility
4. Public comment period — minimum 14 days, longer for breaking changes
5. Revise based on comment
6. Maintainers and Working Group assess against acceptance criteria
7. Accepted RFCs land in the next minor version (backwards-compatible) or next major version (breaking)

### Acceptance criteria for RFCs

- **Multi-vendor support.** At least two structurally independent providers / consumers must indicate they support the change in public comment.
- **No unresolved blocking objections.** A blocking objection requires a documented, substantive concern.
- **Implementation feasibility.** At least one reference implementation must demonstrate the change is implementable.
- **Backwards-compatibility analysis.** Breaking changes require a documented migration path.

---

## 4. Code of conduct

Standard open-source code of conduct applies:

- Be respectful. Technical critique is welcome; personal critique is not.
- Disclose conflicts of interest. If you work for a vendor whose business is materially affected by a proposal you're discussing, say so. Disclosure doesn't exclude participation; it informs other contributors' weighting.
- Use attribution discipline. If you adopt an idea from elsewhere, cite it.
- Don't speak for your employer unless you're authorized to.

Harassment, discrimination, or sustained disruption result in removal from the project. Maintainers are responsible for enforcement and accountable to the community.

---

## 5. The Working Group

The project intends to convene a draft Working Group with multi-vendor representation. Current membership categories targeted:

- Identity providers (LiveRamp, UID2 operators, OpenPass, etc.)
- Context / data providers (Ether Data, content metadata vendors, real-time event APIs)
- Substrate operators (Index Exchange, AWS Advertising, others)
- Consumer-side operators (DSPs, custom-bidding-algo vendors, agency platforms)
- Downstream measurement and attribution platforms
- Publishers and publisher representatives
- Brand-side / agency-side representatives
- Privacy and consent advocates (observer status)

To express interest in Working Group membership, open an issue tagged `working-group` describing your role and interests.

Working Group meetings (when constituted) will be minuted publicly. Decisions follow the RFC process described above.

---

## 6. Licenses for contributions

By submitting a contribution, you agree to license it under the same terms as the rest of the project:

- **Specification text and documentation** under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
- **Schemas and IDL definitions** under [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- **Reference implementations** under [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)

If your contribution is derived from external work, you must have the right to relicense it under these terms, and you should disclose the source.

---

## 7. How to reach out

- **Open an issue** on the repository — the default channel for substantive discussion
- **Email the project maintainer** — for working-group interest, governance questions, or anything you'd rather not open as a public issue initially
- **Pull request** — for direct contributions

If you're representing a vendor or operator and want to coordinate on multi-issue strategic input, an introductory email is welcome.

---

## 8. What the project explicitly wants help with right now

In rough priority order (v0.1):

1. **Working-group formation.** Multi-vendor representation across the categories above. The most important single thing.
2. **LSS drafts for `med_emb` and `tmp_emb`.** The two easiest LSSs to ship in v1.0. Need authors and at least two willing attesters per LSS.
3. **Reference evaluation sets** for the first LSSs. Curated test data with annotations.
4. **JSON Schemas** for the wire format, provenance manifest, and regime taxonomy.
5. **Prebid module** prototype for browser-edge `med_emb` / `phy_emb` emission.
6. **ARTF sidecar reference** for in-loop ORP-conformant ranking inside containerized substrates.
7. **Operational sizing studies.** What does the wire format cost at SSP scale? What's the embedding-store cache architecture that supports pointer-based bidding at national scale?
8. **Regime taxonomy v1.0 draft.** Initial label set per dimension.
9. **Cross-space-relationship LSSs.** `cre_emb × med_emb → brand-safety` is the most consequential.

The maintainer can scope contributions against this list; reach out before starting work on any of the larger items.

---

## See also

- [README.md](README.md)
- [SPEC.md](SPEC.md)
- [docs/06-governance.md](docs/06-governance.md) — formal governance discussion
