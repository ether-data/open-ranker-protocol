# Architecture

This document describes ORP's architectural framework: the four phases of capability the substrate has to support, the three-layer governance model that decides what is universal vs. consumer-specific vs. deployment-local, and the strategic positioning of ORP as a complementary forward prior — not a substitute — for existing ad-tech infrastructure.

Read this document in this order:

1. Layer governance (L1 / L2 / L3) — the cleanest way to answer "what is open and what is proprietary."
2. Forward-prior positioning — where ORP sits relative to incumbents.
3. The four phases of capability — what the substrate has to support.
4. How v1 maps to all of the above.

---

## 1. Layer governance — L1 / L2 / L3

ORP adopts the three-layer governance model used by Ether Data and similar canonical-data stacks. The model gives a crisp, non-arbitrary answer to *what belongs in the open standard* vs. *what stays proprietary*.

### The two-question rule

Every component of the runtime is placed in exactly one of three layers by asking two questions:

1. **Is the source uniform nationally (or globally) with no per-deployment contract?**
2. **Does the component encode downstream-variable history (clicks, conversions, panel response)?**

| L1 — Canonical | L2 — Consumer | L3 — Operational |
| --- | --- | --- |
| Source: uniform | Source: uniform | Source: non-uniform |
| DV history: no | DV history: yes (allowed) | DV history: yes |
| The *form* of the standard | The *use* of the standard | The *deployment* of the standard |

### L1 — the canonical layer

L1 is the universal, deployment-agnostic substrate. It contains:

- The wire format (the OpenRTB extension, the envelope schema).
- The Latent Space Specifications (LSS) — the formal definitions of each first-class embedding space, including conformance gates.
- The closed regime taxonomy.
- The caption-template schema.
- The provenance manifest schema.
- The per-cell × per-time *values* of canonical context inputs (school session, weather observation, transit state, calendar layers), insofar as those inputs are sourced from national feeds (NOAA, GTFS-RT, NCES, federal/state open-data portals).

L1 is what ORP *is*. It is the open standard. It is governed in the open. It does not encode downstream-variable history under any circumstance — the DV-free ablation gate (§12.5 of the SPEC) exists to enforce this.

L1 is also what *every operator can rely on being there*. A buyer running an ORP-conformant ranker can assume L1 is uniform across every substrate it tenants on. A publisher emitting ORP-conformant embeddings can assume any downstream consumer reads the same L1.

### L2 — the consumer layer

L2 is where consumers (DSPs, rankers, custom-algo vendors) consume L1 and build their proprietary decisioning IP on top of it. L2 contains:

- The ranker itself — the learned function combining ORP embeddings into a score.
- The model trained against downstream variables (clicks, conversions, dwell, conversions).
- The consumer's DV-aware feature engineering — `rnf_emb` consumption, conversion-history features, attribution-derived priors.
- The consumer's optional translation layers (Configuration B in §12.3) for unknown-`model_agreement_key` embeddings.

L2 sources its inputs nationally (L1 is uniform) but is allowed to encode DV history because the consumer's job is to predict outcomes against their own attribution. L2 is *proprietary*. It is where competitive differentiation lives among ORP-conformant operators.

### L3 — the operational layer

L3 is deployment-specific: signals that are *not* nationally uniform and that some specific deployment chooses to layer on top. L3 contains:

- Operational live feeds (POS data, sensor counts, signed-contract permits) that exist for a specific commercial deployment but cannot be assumed available everywhere.
- Per-tenant compute envelopes, custom-silicon paths, infrastructure-specific optimizations.
- Operator-specific extensions to the bid request that supplement but do not replace ORP-standard fields.

L3 is *operational*. It is where a specific operator's specific deployment differs from the universal baseline. ORP recognizes L3 exists and does not prevent it; ORP simply does not specify it.

### How layer governance answers "what's open?"

| Question | Layer | Answer |
| --- | --- | --- |
| Wire format and schemas | L1 | Open |
| Latent Space Specifications | L1 | Open |
| Regime taxonomy | L1 | Open (governed via RFC) |
| Caption templates | L1 | Open per-LSS reference, vendor-extensible |
| National canonical-input lookups | L1 | Open (sourced from national feeds) |
| Specific embedding models that conform to LSS | L2 | Proprietary to the embedding provider |
| Ranker that consumes ORP embeddings | L2 | Proprietary to the ranker operator |
| Substrate operator's compute envelope | L3 | Proprietary to the substrate operator |
| Custom silicon path | L3 | Proprietary to the substrate operator |

This is the layering OpenRTB applies implicitly. ORP makes it explicit.

---

## 2. Forward-prior positioning

ORP is a **complementary forward prior**. It sits upstream of existing ad-tech infrastructure as a layer that incumbents can consume — not as an alternative or substitute.

This positioning is deliberate and load-bearing.

### What "forward prior" means concretely

A forward prior is a structured representation of *the moment* — who is here, what is being consumed, where, when, what's around, how the user has been responding — produced upstream of the ranker and delivered into the bid request as ORP-conformant embeddings. Any ORP-aware ranker can consume the prior; any ORP-naive ranker can ignore it; any operator can build a ranker that conditions on it.

The prior is:

- **Computed upstream** of the auction (or at the very beginning of the auction's bid-time inference call).
- **Consumed downstream** by whatever ranker the operator has built.
- **Optional from the ranker's perspective** — fall-through to non-embedding signals is always supported (§12.4).
- **Compatible with** existing identity middleware (RampID, UID2, OpenPass), existing taxonomies (IAB Audience Taxonomy), existing transports (OpenRTB), and existing measurement stacks.

### What "forward prior" replaces

Nothing. ORP doesn't replace OpenRTB; it rides on it. ORP doesn't replace identity middleware; it binds to it (§10 of the SPEC). ORP doesn't replace SSPs or DSPs; it adds a structured prior layer that both sides can consume. ORP doesn't replace measurement vendors; it provides an interpretable prior that measurement stacks can attribute against.

This is not positional modesty. It is the only positioning that actually deploys, because it doesn't require incumbents to lose their existing business in order for the open web to gain a substrate. The first incumbent to consume ORP gains the multi-dimensional ranking capability without disturbing any of their other commercial relationships.

### Why substitute positioning fails

The walled gardens didn't take their position by replacing existing internet infrastructure. They took it by adding capability on top of it that the existing infrastructure couldn't match. Open-web standards bodies that have tried to "replace" incumbent topology (FLoC, various identity-replacement proposals) have run into the same wall: incumbents have no business reason to lose their existing position to a substitute.

A *forward prior* avoids this trap. It is consumable rather than competitive. It compounds the value of existing infrastructure rather than threatening it. Ether Data's commercial team converged on this positioning independently after exposure to the broader topology; ORP adopts it on the same reasoning.

---

## 3. The four phases of capability

ORP is designed against a four-phase architectural target. Phases 1 and 2 ship in v1. Phases 3 and 4 are forward-looking — they explain why ORP is structured the way it is, and they tell operators what to leave architectural room for.

The phases are not implementation milestones for any specific operator. They are *layers of capability* the substrate has to support. A v1-compliant implementation supports Phases 1 and 2; a fully evolved deployment runs all four.

### Phase 1 — Context Foundation

**Goal:** a continuously refreshed multimodal latent representation of the environment around an impression opportunity.

This is the layer the open web is furthest behind on. Walled-garden rankers consume rich, multimodal context as a first-class input set. The open web has historically passed page-level metadata and called it context.

Phase 1 defines the **context envelope** — a structured object carrying multiple distinct context signals together, each as its own embedding, each with its own provider, each with its own provenance. The envelope spec is in [docs/03-context-envelope.md](03-context-envelope.md).

The signals carried in the envelope include (drawing on Ether Data's E_moment proposal and the broader collaborator framing):

- H3 spatial structure (Uber's hexagonal grid; R8 default)
- Temporal dynamics (cyclical encoding, recency, dwell, seasonality)
- POI composition
- Census / socioeconomic priors
- Calendar layers (religious, civic, academic, sports, cultural)
- Transit and mobility state
- Weather observation
- Permits and live events
- Content / page metadata
- Device and inventory metadata
- Surrounding creative

Each signal can come from a different provider. The envelope is the package; the LSS framework defines what each space means. Multi-provider composition is the design.

### Phase 2 — Cross-Modal Alignment

**Goal:** separate encoders for context, creative, campaign intent, and response history, projected into compatible latent spaces that a ranker can interact with jointly.

Phase 1 produced multimodal context. Phase 2 makes it *combinable* with the other inputs to a ranker — the user side (identity, reinforcement), the buyer side (creative, audience query), the context bundle from Phase 1.

The architectural commitment in Phase 2: **don't collapse anything into a single fused vector before the ranker sees it**. Each input keeps its own latent space; the ranker learns how to combine them.

This is what walled-garden multi-tower / multi-input ranking architectures do. The ranker has separate input pathways per embedding type. Combination is learned end-to-end against the ranking objective.

What Phase 2 needs from ORP:

- Separate latent-space definitions per embedding type (the LSS framework — see [docs/02-latent-space.md](02-latent-space.md)).
- A wire format that delivers multiple embeddings together (SPEC §9).
- Provenance manifests that declare conformance to LSS (SPEC §11.3).
- Cross-space relationships expressible in the LSS framework (e.g., `cre_emb × med_emb → brand-safety score`).

Phase 2 is the architectural commitment to multi-embedding ranking. Phase 1 is the structured payload; Phase 2 is the discipline that makes the payload composable.

### Phase 3 — World-State Modeling (v2)

**Goal:** a latent state prediction layer that infers missing signals, predicts future contextual states, and models transitions between states.

Phase 3 is the most forward-looking. Phases 1 and 2 produce embeddings the ranker can score. Phase 3 produces a *predictive latent state* the ranker can reason against.

Production walled-garden recommendation systems are evolving past static-similarity ranking toward models that:

- Predict the next event in a user's behavioral sequence (e.g., Meta HSTU).
- Infer missing context when some signals are absent (e.g., what's likely happening at a place given POI + time + weather).
- Project future state from current state (e.g., will this user be in-market next week given current trajectory).
- Model state transitions across a session.

ORP's Phase 3 supports this via the optional `wst_emb` slot (SPEC §8) — a world-state embedding produced by a predictor that sits between the encoders and the ranker.

```
Encoders → World-state predictor → Ranking model → Score
```

The predictor is not the scorer. It produces an enriched latent representation that the ranker then uses.

ORP does not mandate JEPA, sequence learning, or any other specific architecture for the predictor. The slot is architecture-agnostic. Operators can fill it with whatever predictor produces the best downstream lift while passing the conformance gates the LSS for `wst_emb` will specify.

Phase 3 is v2 because the standardization moment isn't ripe at the walled-garden side yet — Meta's HSTU paradigm is months old as of mid-2026 and is still evolving. v1 keeps the architectural slot but doesn't require it.

### Phase 4 — Ranking + Reinforcement

**Goal:** bid-time inference combining all inputs.

```
score = f(uid_emb, context_envelope, cre_emb, aud_emb, rnf_emb, [wst_emb])
```

Phase 4 is what the operator (DSP, bidder, ranker) actually runs. **ORP does not specify the ranker.** The ranker is L2 in the layer model — proprietary to the operator. ORP specifies the inputs, the latent-space semantics, the wire format, the provenance, and the interpretability surface.

Common ranker architectures operators run at Phase 4:

- Gradient boosting over flattened embedding features — simple, interpretable, good baseline.
- Shallow neural rankers with embedding-aware input layers — closer to walled-garden style at lower cost.
- Multi-tower architectures — separate towers per input dimension, learned interaction layer (the DLRM lineage).
- Cross-encoder rankers — embeddings combined directly without prior projection to a shared space.
- Sequence-aware rankers (HSTU-class, v2) — consume event sequences for user-side input.
- Online / adaptive rankers — update weights from real-time reinforcement signals.

What ORP requires of any Phase 4 ranker:

1. Consume embeddings according to their declared LSS conformance.
2. Honor consent and TTL declarations from each embedding's provenance manifest.
3. Optionally log decision rationale in the BidResponse extension for interpretability ([docs/05-interpretability.md](05-interpretability.md)).
4. Publish a ranker manifest declaring which embedding inputs it consumes, what LSS versions it expects, and what its compute envelope is (for substrate scheduling, if running on shared substrate infrastructure).

Inside those constraints, the ranker is L2 — operator IP.

---

## 4. How v1 maps to all of this

| Phase / layer | v1 deliverable | v2+ extension |
| --- | --- | --- |
| Phase 1 (Context Foundation) | Context envelope spec; 5 context-suite embeddings; composite-embedding support; H3 R8 default | Additional context signals (mobility class, attention priors, real-time event taxonomy) |
| Phase 2 (Cross-Modal Alignment) | LSS framework with 5 conformance gates; cross-space relationship declarations | Reference foundation models for each space; richer cross-space LSS |
| Phase 3 (World-State Modeling) | Architectural slot only; `wst_emb` reserved in v2 roadmap | `wst_emb` first-class type; world-state predictor LSS; sequence-learning support via `evt_seq_emb` |
| Phase 4 (Ranking + Reinforcement) | Ranker manifest contract; decision-disclosure interface; compute-envelope declaration | Standardized adaptive-ranker conformance |
| L1 (Canonical) | Wire format, LSS framework, regime taxonomy v1.0, caption template schema, provenance manifest schema, national input lookups | Foundation-model candidates; LSS for v2 spaces |
| L2 (Consumer) | Ranker manifest contract | Adaptive ranker conformance |
| L3 (Operational) | Not specified (operator domain) | Not specified (operator domain) |

v1 ships Phases 1 and 2 with the L1/L2 boundary clearly drawn. v2 fills in Phase 3 and the more demanding LSS work. L3 is and remains the operator's domain.

---

## 5. Why this architecture, not another

Three alternative architectures the project considered and rejected, with the reasons.

### Alternative A — single-embedding standard

The simplest possible standard: one user-side embedding, one buyer-side embedding, match by cosine similarity. *Rejected* because it doesn't approach walled-garden parity (which is multi-input ranking) and because it forecloses the most valuable contribution — making *context as a suite* a first-class concept.

### Alternative B — operator-defined latent spaces

Each operator publishes its own latent spaces and brings them to its tenants. *Rejected* because it collapses to one operator's model dominating, and because it provides no cross-vendor portability without bilateral translation layers. ORP's value-add is the *shared semantic ground*; operator-defined spaces don't provide that.

### Alternative C — single composite "moment" embedding

Treat each impression as a single fused composite embedding produced by one provider and consumed as-is. *Rejected* because it conflates input signals from different sources into one vector that loses the multi-provider compositionality the open web needs. ORP allows composite embeddings (§6.7) but does not require them — providers who emit elemental embeddings remain first-class.

The chosen architecture — layered governance, forward-prior positioning, four phases of capability — is the one that makes the standard *deployable today* (Phase 1 + Phase 2 ship in v1, L1/L2 boundary is clean) and *forward-evolvable* (Phase 3 and Phase 4 leave architectural room without requiring v1 to wait on them).

---

## See also

- [docs/02-latent-space.md](02-latent-space.md) — the LSS framework that makes Phase 2 work
- [docs/03-context-envelope.md](03-context-envelope.md) — Phase 1 envelope specification in detail
- [docs/04-provider-integration.md](04-provider-integration.md) — how providers plug into each phase
- [docs/05-interpretability.md](05-interpretability.md) — the caption / regime / audit surface
- [docs/06-governance.md](06-governance.md) — the open vs. proprietary boundary across all four phases
- [docs/07-runtime-model.md](07-runtime-model.md) — what happens at bid time across all four phases
- [SPEC.md](../SPEC.md) — formal wire-format specification
