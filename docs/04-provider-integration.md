# Provider Integration

This document describes how a provider — an embedding vendor, a data provider, an identity middleware, a context platform, an ad server — integrates their proprietary models with the ORP standard so their embeddings can be consumed by any ORP-conformant ranker.

The contract is deliberately permissive: ORP defines *what* a conformant embedding looks like, not *how* the provider produces it. The provider keeps their model, their training data, their infrastructure, and their commercial pricing proprietary. They commit only to:

1. **The wire format.** Embeddings emitted into the bid request match SPEC §9.
2. **The provenance manifest.** Every embedding ships with a complete manifest per SPEC §11.3.
3. **The interpretability surface.** Every embedding ships with a runtime caption and regime label per SPEC §6.5–§6.6.
4. **The conformance gates.** Models claiming LSS conformance pass the published gates per SPEC §12.5.

Once those four are in place, the provider's embeddings are interoperable with any ORP-conformant downstream consumer.

---

## 1. What providers integrate

ORP recognizes several distinct provider types, each plugging into a different position in the runtime. Most providers fit one of these patterns; some fit several.

| Provider type | What they emit | Typical examples |
| --- | --- | --- |
| Identity provider | `uid_emb` | LiveRamp (ATS), UID2 operators, Open Pass, first-party publishers, retailers |
| Context-content provider | `med_emb` | Publishers, content-metadata vendors (Gracenote-class), CMS platforms |
| Geospatial / place provider | `phy_emb`, contributions to composite context | Ether Data, geo-signal vendors, mobility-data vendors, POI graphs |
| Temporal-reference provider | `tmp_emb` | ORP reference encoder (deterministic, open-source); operators may also emit enriched versions |
| Household / social provider | `soc_emb` | Identity middleware with household graphs, retailer co-purchase graphs |
| Real-world event provider | `rwc_emb` | Sports data, news APIs, weather APIs, civic open-data providers |
| Creative provider | `cre_emb` | Creative agencies, DSP creative services, ad servers |
| Audience-query provider | `aud_emb` | DSPs, custom-bidding-algo vendors (Chalice / Scibids / Empowered), agency planning tools |
| Reinforcement provider | `rnf_emb` | Measurement vendors, DSP feedback services, retailer post-purchase signal |
| Composite-context provider | Composite `med_emb` or new composite type | Ether Data E_moment, future entrants offering joint place × time × world-state |

A single commercial entity may be several of these — e.g., LiveRamp emits `uid_emb` and `soc_emb`; Ether Data emits a composite context embedding and likely contributes to `phy_emb` and `rwc_emb` as well.

---

## 2. The integration steps

### Step 1 — Choose the LSS path or the model-agreement path

Two coordination paths exist (see [docs/02-latent-space.md](02-latent-space.md) §2):

- **LSS conformance** — train a model whose embeddings conform to a published Latent Space Specification. Pass the LSS's gates. Get cross-vendor interop without bilateral arrangements. Recommended for any provider with multi-customer ambitions.
- **Bilateral model agreement** — agree with a specific consumer on a `model_agreement_key`. Simpler for one-off integrations or when no LSS yet exists for the space.

Most v1 providers will be on the bilateral path initially (because LSSs for some spaces are still in draft) and transition to LSS conformance as the LSSs mature.

### Step 2 — Implement the wire format

Embeddings emit into the bid request's `ext.orp.embeddings` array, matching SPEC §9. Required fields per embedding:

- `type` — one of the first-class types (`uid_emb`, `med_emb`, `phy_emb`, etc.)
- `source_id` — unique identifier for the provider's model
- `model_agreement_key` — opaque identifier naming the (model, version, training-objective, training-snapshot) tuple
- `dim`, `dtype`, `norm` — geometric properties
- `vector` — the embedding itself, base64-encoded compact binary
- `runtime_caption` — deterministic plain-language rendering of the inputs (see Step 4)
- `regime_label` — categorical label from the regime taxonomy (see Step 4)
- `provenance_uri` — pointer to the provenance manifest (see Step 3)
- `composite_inputs` — if a composite, the list of input pipelines consumed
- `spatial_binding` — if place-scoped (e.g., `phy_emb`), the H3 cell ID and resolution

### Step 3 — Publish the provenance manifest

The provenance manifest is the contract for what the embedding represents. It's hosted at `provenance_uri` and includes (SPEC §11.3):

- `embedding_type` and `embedding_class` (elemental vs. composite)
- `model_lineage` — architecture, training data, training objective
- `data_provenance` — what populations the embedding was derived from
- `inference_class` — `measured`, `inferred`, or `probabilistic`
- `consent_purposes`, `jurisdictional_scope`
- `caption_template_uri` — the deterministic template used to render runtime captions
- `regime_taxonomy_uri` — the closed taxonomy this embedding's labels are drawn from
- `activation_rate` — fraction of impressions where the embedding fires
- `null_semantics` — how to interpret missing instances
- `population_excluded` — explicitly excluded populations (e.g., children, sensitive categories)
- `update_cadence` — how often the embedding is recomputed
- `conformance_attestations` — per-gate attestation if claiming LSS conformance
- `attestation` — cryptographic signature from the provider over the manifest

The provenance manifest is read by both machines (downstream rankers, conformance auditors) and humans (compliance reviewers, brand legal teams). The schema is open; the content reflects what the provider is willing to commit to publicly.

### Step 4 — Implement the caption template and regime taxonomy

For every embedding the provider emits, two interpretability artifacts ride alongside:

**Runtime caption** — a deterministic, templated sentence rendered from the same canonical inputs the embedding consumed. The template is non-neural (slot-filling), faithful by construction, and published at `caption_template_uri`. Example for a composite context embedding:

```
{daypart} {dow}, {neighborhood_name}, {weather_summary}.
{calendar_phrase}. {school_phrase}. {disruption_phrase}. {event_phrase}.
{poi_active_phrase}. {regime_phrase}.
```

Rendered for one impression: *"Wednesday afternoon, Morningside Heights, 22°C clear. Columbia in move-in week. PS 36 in session, dismissal at 14:50. No transit disruption. Block-party permit on 115th St. Restaurants, grocers, citibike active."*

**Regime label** — a discrete categorical label drawn from a published closed taxonomy. The provider chooses which label applies to each instance from the taxonomy's enumerated list. Example: `context.university-move-in-spike`.

Both are *generated upstream of (or in parallel with) the embedding encoder*, from the same input table. They are not post-hoc explanations of the vector — they are co-publications that the encoder also consumes.

### Step 5 — Pass conformance gates (if claiming LSS conformance)

If the provider is claiming LSS conformance for one or more embedding types, they run the published evaluation against the LSS's reference test set and publish attestations per gate (SPEC §12.5). The attestations are reproducible: any consumer can re-run the evaluation and verify the claim.

Conformance is not a one-time certification. LSSs typically require re-attestation per major model version or per quarter, whichever is more frequent.

### Step 6 — Deploy through a substrate or directly

The provider's embeddings reach the ranker via one of three deployment patterns:

- **Direct emission in the bid request.** The provider integrates with the SSP / exchange and embeddings ride in `ext.orp.embeddings` for every impression the provider is configured to enrich.
- **Pointer-based with embedding store.** For durable, cacheable embeddings (`uid_emb`, `phy_emb` per cell), the provider populates a shared embedding store; the bid request carries pointers; the bidder resolves them at inference time. This is the operational pattern at national scale (SPEC §9.4, [docs/03-context-envelope.md](03-context-envelope.md) §7).
- **Substrate co-tenancy.** The provider runs as an ARTF-class container co-located with the SSP's infrastructure. Embeddings are computed in-environment and emitted into the bid request without leaving the substrate. This is the highest-performance pattern and the one walled-garden-parity architectures target.

The deployment pattern is the operator's choice. ORP supports all three.

---

## 3. The provider commitment (in one paragraph)

A provider committing to ORP is committing to: *we will emit our embeddings into ORP-conformant bid requests with full provenance, with runtime captions and regime labels, with declared activation rates and null semantics, optionally with LSS-gate attestations, and we will keep our model agreement key stable across the lifetime of each model version. In exchange, our embeddings become readable by any ORP-conformant downstream consumer without bilateral integration work.*

The exchange is symmetric: consumers commit to honoring the provenance manifest's consent and jurisdiction declarations, to logging their decisions in the decision-disclosure interface where the embedding contributed, and to attributing the embedding correctly in any measurement loopback.

---

## 4. What providers retain

ORP is deliberately silent on most of what makes a provider's offering valuable:

- **The model itself.** Architecture, training procedure, training data, training infrastructure — all proprietary.
- **The encoder code.** Open-sourcing the encoder is voluntary, not required.
- **The pricing.** Per-impression, per-query, subscription, revenue-share — provider's choice.
- **The integration go-to-market.** Direct sales, channel partner, embedded in a substrate operator's offering — provider's choice.
- **The product surface.** A provider can sell the embeddings as a product, the rankings as a product, an API as a product — whatever fits their commercial topology.

The standard governs the *interface*. The *product* on either side of the interface is the operator's IP. This is the same pattern OpenRTB applied to programmatic and the broader internet stack applies to networked services generally.

---

## 5. A worked integration — Ether Data's E_moment

Illustrative end-to-end integration of a real provider against the ORP spec, drawn from [KB §19](../../knowledge-base/19-ether-emoment-proposal.md).

### What Ether Data emits

A composite `med_emb` (initially; potentially elevated to its own composite type in v2) that captures the joint state of an H3 R8 cell at an hour-level moment under live context.

### Integration components

| Component | What it is | How it integrates |
| --- | --- | --- |
| Encoder | The E_moment model (initially bilinear cross of sensitivity priors × context activation; future JEPA pilot) | Trained on canonical inputs; emits 128-dim float16 L2-normalized vector per cell-hour |
| `composite_inputs` | `["place_h3r8", "calendar_layers", "weather_observation", "school_session", "transit_state", "permits", "poi_activity"]` | Declared in provenance manifest |
| Caption template | Deterministic slot-filling template ([Ether's spec §5](../../knowledge-base/19-ether-emoment-proposal.md#5-descriptivity--the-moment-caption-decoder)) | Hosted at `caption_template_uri`; renders per-instance captions from same inputs as encoder |
| Regime taxonomy | Closed taxonomy of context regimes (`context.university-move-in-spike`, `context.transit-disrupted-weekend`, `context.holiday-iftar-window`, etc.) | Versioned, published at `regime_taxonomy_uri` |
| Conformance gates | All five SPEC §12.5 gates: DV-free ablation, conditional signal density, cross-metro generalization, spatial equity, cell-mask ablation (for JEPA variants) | Run quarterly against the LSS reference set; results published as `conformance_attestations` |
| Spatial binding | H3 R8 cell ID | Carried in `spatial_binding.cell_id` |
| Activation rate | Variable per signal; ~0.02 for Ramadan-active cells × hours, ~0.95 for the always-on physical baseline | Declared per signal in the manifest |

### Deployment pattern

Substrate co-tenancy is Ether's commercial story — E_moment runs inside ARTF-class container environments at SSPs / exchanges that want to enrich bid requests with composite context. Pointer-based deployment for downstream consumers that prefer to resolve from an embedding store.

### What Ether retains

- The encoder model (committee chose bilinear-cross for canonical, JEPA for research).
- The training pipeline and data.
- The forecasting and prediction APIs (commercial moat).
- The world-state predictor variants (Phase 3 territory; proprietary).
- The pricing model.
- The customer relationships.

### What Ether commits to the standard

- Wire format conformance.
- Provenance manifest publication.
- Caption template and regime taxonomy published.
- Conformance gate attestation quarterly.
- Stable `model_agreement_key` per model version.

This is the deal. The standard gets interoperability; the provider gets ecosystem reach without surrendering their model.

---

## 6. The provider on-ramp

Practical path for a provider considering ORP integration:

1. **Read SPEC.md and the relevant LSS document(s).** Understand what conformance means for the specific embedding type(s) you'd emit.
2. **Prototype against the wire format.** Generate sample bid-request extensions with your embeddings. Verify they parse against the JSON Schema (in `schemas/` when published).
3. **Publish a draft provenance manifest.** Document what your model represents and how it was trained. Don't need to commit to LSS gates yet.
4. **Get one bilateral consumer running on `model_agreement_key`.** Bypass the LSS gates initially; just establish that the embeddings flow correctly to a real ranker.
5. **Decide whether to pursue LSS conformance.** Once an LSS for your embedding type is published, evaluate against the gates. Attest if you pass. Iterate if you don't.
6. **Engage the working group.** RFC-style proposals are how the standard evolves. Providers with operational experience are the most important voices in shaping LSSs.

Steps 1-4 can happen in weeks. Step 5 is the longer-cycle work. Step 6 is continuous.

---

## See also

- [docs/02-latent-space.md](02-latent-space.md) — LSS framework, conformance gates
- [docs/03-context-envelope.md](03-context-envelope.md) — multi-provider envelope structure
- [docs/05-interpretability.md](05-interpretability.md) — caption + regime label as provider-published artifacts
- [docs/06-governance.md](06-governance.md) — open / proprietary boundary
- [SPEC.md §9](../SPEC.md#9-wire-format) — wire format
- [SPEC.md §11](../SPEC.md#11-privacy-consent-and-audit) — provenance manifest, consent, audit
- [SPEC.md §12](../SPEC.md#12-model-interoperability) — model interoperability, conformance gates
- [KB §19 Ether Data E_moment](../../knowledge-base/19-ether-emoment-proposal.md) — worked-example integration
