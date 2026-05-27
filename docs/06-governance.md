# Governance

This document describes the governance model of ORP: what is open and what is proprietary, where the boundary is drawn and why, how the standard evolves, and where it intends to live long-term.

The governance design has one anchor commitment: **ORP is a complementary forward prior, not a substitute.** Everything else follows from that. The open / proprietary boundary, the evolution process, the standards-body path — all are downstream of this positioning.

---

## 1. The forward-prior positioning

ORP sits upstream of existing ad-tech infrastructure as a layer that incumbents can consume. It does not replace OpenRTB; it rides on it. It does not replace identity middleware; it binds to it. It does not replace SSPs or DSPs; it adds a structured prior layer both sides can consume. It does not replace measurement vendors; it provides an interpretable prior that measurement stacks can attribute against.

This positioning is not modesty. It is the only positioning that actually deploys, because it doesn't require incumbents to lose existing business in order for the open web to gain a substrate. The first incumbent to consume ORP gains multi-dimensional ranking capability without disturbing any of their other commercial relationships.

This is the same positioning Ether Data's commercial team converged on independently and codified in their committee review (KB §19, [Nate-at-Ether's verdict](../../knowledge-base/19-ether-emoment-proposal.md#nate-founder--commercial-lead)). It is documented here as an explicit governance commitment so the project doesn't drift into substitute framing later.

### What "complementary forward prior" implies for governance

- ORP standards work **does not displace** OpenRTB, AAMP, OpenDirect, or any other IAB Tech Lab artifact. ORP composes with them.
- ORP standards work **does not require** a single foundation model, a single substrate operator, or a single identity provider to win the market.
- ORP standards work **does require** multi-vendor participation. The standard's value is interoperability across providers; governance must therefore prioritize multi-vendor representation over any single contributor's preferences.

---

## 2. The open / proprietary boundary

The simplest statement of the boundary:

> **Interfaces are open. Implementations are proprietary.**

This is the same pattern OpenRTB applied to programmatic, that the broader internet stack applies to networked services, and that successful commercial-and-open ecosystems consistently use.

### What's open (governed by this project)

| Category | Open artifact |
| --- | --- |
| Wire format | SPEC.md — OpenRTB extension, envelope schema, payload structure |
| Semantic ground | Latent Space Specifications (LSS) — what each embedding type means; conformance gates; reference evaluation sets |
| Interpretability | Closed regime taxonomy; caption template schema; provenance manifest schema |
| Identity binding | Mapping rules between embeddings and existing identifier systems (RampID, UID2, OpenPass, etc.) |
| Privacy & consent | Consent declarations, jurisdictional scope, erasure endpoints, audit attestations |
| Conformance | Five-gate evaluation framework; reference evaluation sets per LSS; reproducible audit methodology |

These are all CC BY 4.0 (specification text) and Apache 2.0 (reference implementations).

### What's proprietary (operator's domain)

| Category | Proprietary artifact |
| --- | --- |
| Embedding models | The specific encoders that produce embeddings — architecture, training data, training procedure, training infrastructure |
| Training data | The data the model was trained on |
| Inference infrastructure | The substrate the model runs on, including custom silicon paths |
| Ranking models | The L2 ranker that consumes ORP embeddings |
| World-state predictors | Phase 3 prediction models (JEPA-class or other) |
| Customer integrations | The commercial agreements between providers and consumers |
| Pricing | Per-impression, per-query, subscription, revenue-share — provider's choice |
| Product surfaces | Specific products built on ORP-conformant embeddings |

These are the operator's IP. ORP is silent on them by design.

### Why this boundary is the right one

Three principles inform the placement:

1. **Where bilateral relationships dominate, leave it proprietary.** Embedding models, ranking models, customer integrations are bilateral in nature — provider and consumer choose what fits their commercial topology. Standardizing them would over-constrain the market.
2. **Where multilateral coordination is required, make it open.** Wire format, semantic ground, conformance gates, interpretability surfaces are all multilateral — they only work if everyone agrees. Open governance is the only viable path.
3. **Where audit pressure is highest, make it open and reproducible.** Conformance gates, regime taxonomy, provenance manifests must be auditable by regulators and compliance teams. Closed-source audit is not audit.

These principles draw clean boundaries. They also draw *defensible* boundaries — a provider considering whether to commit to ORP can read this document and verify their commercial moat is left intact.

---

## 3. The provider commercial model

ORP exists to enable a specific commercial pattern: **providers sell proprietary models that target an open standard.** The open standard creates ecosystem-scale interoperability and adoption; the proprietary models capture the value of the inference and decisioning layer.

Examples of viable provider commercial surfaces that target ORP:

- **Embedding APIs.** Per-impression or per-query pricing for embeddings emitted into ORP-conformant bid requests.
- **Substrate co-tenant services.** Provider runs as an ARTF-class container at SSP / exchange infrastructure; the substrate operator charges; the provider charges; everyone gets paid.
- **World-state prediction APIs.** Phase 3 predictors sold as an enrichment layer that consumers integrate with their ranker.
- **Creative-context fit scoring.** Brand-safety / suitability scores derived from cross-space relationships (`cre_emb × med_emb`), sold as an API.
- **Forecasting and reinforcement models.** Longer-horizon predictions sold to MMM, planning, or attribution platforms.
- **Conformance attestation services.** Third-party audit of LSS conformance, sold to providers and to consumers requiring third-party validation.

The open standard drives interoperability and adoption. The proprietary models are the paid inference layer that powers optimization and decisioning. Both layers thrive when the boundary is clean.

This is the commercial frame Laurent Madeleni articulated in the original collaborator thread:

> *"We could open source / give away to the IAB a standardized 'context envelope' specification for bid-time AI systems. Not the models themselves, but the interoperable payload format... And we monetize the intelligence layer itself: context embedding APIs, bid-time ranking APIs, prediction/scoring APIs, latent world-state APIs, creative-context fit scoring, forecasting and reinforcement models. The open standard drives interoperability and adoption, while the proprietary models become the paid inference layer powering optimization and decisioning."*

ORP's governance is designed to make this commercial pattern viable for any provider who can build the underlying models.

---

## 4. Standards-body path

The project's intended long-term home is a neutral standards body. The first preference is **IAB Tech Lab** as an addition to the AAMP framework, alongside ARTF, Agentic Audiences, and the buyer/seller agent SDKs.

The reasoning:

- **AAMP is the natural neighborhood.** ARTF defines the execution substrate; Agentic Audiences defines a precursor of the embedding-exchange concept; ORP defines the multi-input ranking inputs and the latent-space ontology. Together they're a coherent stack.
- **IAB Tech Lab has the convening power.** The relevant providers (LiveRamp, Index Exchange, AWS, agency platforms, ad servers) are members. Working group formation is operationally feasible.
- **Multi-vendor governance is the norm there.** ORP's value depends on multi-vendor participation; IAB's RFC process matches this requirement.

Alternative paths considered:

- **W3C** — possible if ORP expands toward browser-API integration. Currently out of scope.
- **Independent foundation** — possible if a coalition of operators prefers operator-led governance independent of IAB.

Until the project finds its permanent home, governance proceeds as described in §5 below — RFC-based, open, multi-vendor.

---

## 5. How the standard evolves — the RFC process

Changes to SPEC, to any LSS, or to the regime taxonomy follow an RFC-style proposal process. Steps:

1. **Issue or discussion** — author raises a question or proposes a change on the repository. Discussion gathers community input.
2. **RFC draft** — author writes a formal RFC: motivation, design, alternatives considered, migration path, impact on existing implementations.
3. **Public comment period** — minimum 14 days. Affected parties (providers, substrate operators, consumers, downstream measurement) weigh in.
4. **Revision** — author revises based on comment.
5. **Acceptance or rejection** — formal acceptance criteria (see below) determine outcome.
6. **Implementation and shipping** — accepted RFCs land in the next minor version (for backwards-compatible changes) or next major version (for breaking changes).

### Acceptance criteria

For an RFC to be accepted, it must satisfy:

- **Multi-vendor support.** At least two structurally independent providers / consumers must support the change in public comment.
- **No unresolved blocking objections.** A blocking objection requires a documented, substantive concern; consensus is preferred but not always achievable.
- **Implementation feasibility.** At least one reference implementation must demonstrate the change is implementable at acceptable cost.
- **Backwards-compatibility analysis.** Breaking changes require a documented migration path with reasonable timeline.

### Versioning

| Type | Example | Backwards-compatible? |
| --- | --- | --- |
| Patch (`v1.0.1`) | Typo, clarification | Yes |
| Minor (`v1.1`) | New fields, new LSS, new regime labels | Yes |
| Major (`v2.0`) | Wire format change, semantic shift in an LSS | No (migration required) |

Each version is stable: an operator using `v1.0` is immune to changes in `v1.1` and later until they explicitly upgrade.

---

## 6. The conformance certification model

ORP supports multiple levels of conformance attestation, recognizing that different deployment contexts require different levels of rigor.

### Self-attestation (default)

A provider runs the published conformance evaluation against their model and publishes the attestation in their provenance manifest. The attestation is reproducible: any consumer can re-run the evaluation against the published reference set.

Self-attestation is the v1 baseline. Most providers and consumers will operate at this level.

### Third-party attestation (regulated verticals, v2 candidate)

For regulated verticals (healthcare, financial services, alcohol, political, children's products), some deployments may require third-party attestation of conformance. The third party (an auditor) re-runs the published evaluation, signs the result, and the signed attestation is referenced in the provenance manifest.

ORP v1 supports third-party attestation as an option; v2 may make it mandatory in specific verticals.

### Continuous attestation

LSS conformance is not a one-time certification. Attestations expire on a published cadence — typically per major model version (provider re-attests after retraining) or per quarter (whichever is more frequent). Stale attestations are visible in the provenance manifest as expired, which downstream consumers can act on.

---

## 7. Licenses

ORP's open artifacts are released under permissive licenses:

| Artifact category | License |
| --- | --- |
| Specification text (SPEC.md, docs/*.md, lss/*) | [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) |
| JSON Schemas, OpenRTB extension definitions, gRPC IDL | [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) |
| Reference SDKs (TypeScript, Python), reference encoders, reference templates | [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) |
| Reference evaluation datasets | Per-LSS, but permissive (CC BY or equivalent); reuse with attribution |

CC BY 4.0 allows any party — including commercial vendors — to redistribute, modify, and build on the specification with attribution. Apache 2.0 grants explicit patent rights and allows commercial use of the reference implementations without onerous redistribution requirements.

The licenses are chosen to maximize ecosystem adoption velocity. Restrictive licenses (copyleft, non-commercial, share-alike) were considered and rejected because they create friction at the point where a commercial provider integrates the standard, which is the wrong friction for a standard whose value is adoption.

---

## 8. Contributors and the Working Group

The project intends to convene a **draft Working Group** during v0.1 development, with the explicit goal of multi-vendor representation:

- Providers (identity, context, creative, audience)
- Substrate operators (SSPs, exchanges, hyperscaler ad-tech)
- Consumer-side operators (DSPs, custom-algo vendors)
- Downstream measurement and attribution platforms
- Publishers and publisher representatives
- Brand-side / agency-side representatives
- Privacy and consent advocates / regulators (observer status, not vote)

The Working Group operates on the RFC process described above. Working Group decisions are documented publicly; Working Group meetings are minuted publicly.

Interested parties express interest by opening an issue on the repository or by emailing the project maintainer.

### Code of conduct

Standard open-source code of conduct applies: respectful discourse, no harassment, technical critique welcome and personal critique not, attribution discipline, conflict-of-interest disclosure for substantive contributions.

---

## 9. Conflicts of interest and impartiality

ORP's value proposition depends on impartiality. Specific commitments:

- **No single provider's interest overrides the standard's interoperability.** If an RFC would benefit one provider at the cost of multi-vendor interop, it is rejected regardless of how active that provider is in the Working Group.
- **No proprietary model gets named in normative spec text.** The spec references *categories* of provider (identity middleware, context provider, etc.) without endorsing specific vendors. Reference implementations and worked examples in docs may name specific providers for illustration, but normative requirements never do.
- **Conformance gate thresholds are published, not negotiated bilaterally.** An LSS gate threshold is the same for every provider attesting against it.
- **Contributors disclose commercial interests** when proposing or reviewing RFCs that bear on those interests. Disclosure does not exclude participation; it informs other contributors' weighting.

---

## 10. What the governance design is optimizing for

The end state ORP's governance design is targeting:

> A multilateral standards body governs the interfaces. Many proprietary vendors compete on the implementations. Substrate operators amortize custom-silicon and infrastructure investment across many tenants. Consumers (DSPs, agencies, brands) get cross-vendor interoperability without bilateral integration work. Regulators get auditable interpretability surfaces. The open web closes the substrate gap to walled gardens without ceding its multi-vendor structure to any single operator.

This is what "complementary forward prior" makes operationally possible. Every governance choice in this document — license, RFC process, conformance model, standards-body path — serves this end state.

---

## See also

- [docs/01-architecture.md](01-architecture.md) — the four phases and L1/L2/L3 layer model that the governance boundary draws on
- [docs/02-latent-space.md](02-latent-space.md) — LSS framework, the most consequential open artifact
- [docs/04-provider-integration.md](04-provider-integration.md) — how providers commit to the standard while retaining their commercial IP
- [docs/05-interpretability.md](05-interpretability.md) — interpretability surfaces, audit, regulator readability
- [SPEC.md §18](../SPEC.md#18-governance) — formal governance section in the specification
- [KB §19 Ether Data E_moment](../../knowledge-base/19-ether-emoment-proposal.md) — origin of the forward-prior positioning
- [KB §10 strategy assessment](../../knowledge-base/10-strategy-assessment.md) — strategic context for why this governance posture matters
