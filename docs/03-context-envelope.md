# The Multimodal Context Envelope

This document specifies the structure of the context payload that rides in an ORP-conformant bid request — the bundle of signals describing the moment around an impression opportunity, drawn from many providers, assembled into a single envelope, and consumed jointly by the ranker.

The envelope is the operational expression of Phase 1 ([docs/01-architecture.md](01-architecture.md)). The LSS framework ([docs/02-latent-space.md](02-latent-space.md)) defines what each signal in the envelope means. This document defines how they travel together.

---

## 1. Why a context envelope, not a single embedding

The simplest alternative would be: one provider emits one "context embedding" per impression, and the ranker consumes it. We don't do that, and the reasons are worth stating clearly.

### Context is multi-source by nature

A publisher knows the page. A geospatial vendor knows the place. A weather API knows the conditions. A real-time event provider knows what's happening in the world. No single vendor knows all of these and no single vendor should be forced to. The envelope is the *composition mechanism* that lets each provider contribute the signal they're best at.

### Context is multi-facet by nature

What's being consumed (media) is a different question from where the consumption is happening (physical), from when (temporal), from who else is around (social), from what's happening in the world (real-world). Collapsing these into one vector before the ranker sees them loses the multi-dimensional structure the ranker is supposed to exploit. The envelope preserves the structure.

### Context is multi-provenance by nature

A regulator auditing a winning bid asks "what did the ranker see and where did it come from?" An envelope with separately-provenanced embeddings answers that question per signal. A single fused vector does not.

These three reasons — multi-source, multi-facet, multi-provenance — are the design force behind the envelope structure.

---

## 2. Envelope structure

The context envelope is the part of the bid request's `ext.orp.embeddings` array that contributes context-suite embeddings. v1 carries up to five distinct context dimensions:

| Dimension | Type | Description |
| --- | --- | --- |
| Media | `med_emb` | What is being consumed |
| Physical | `phy_emb` | Where and how |
| Temporal | `tmp_emb` | When |
| Social | `soc_emb` | Who is around |
| Real-world | `rwc_emb` | What is happening in the world |

The envelope:

- Carries *zero or more* instances of each dimension. A bid request may have one `med_emb` (from the publisher) and two `phy_emb` (one from the device, one from a geo-signal provider); the ranker chooses among them.
- Carries each instance with its full provenance (SPEC §11.3) — model lineage, training data, consent, conformance attestations.
- Carries each instance with its runtime caption and regime label (SPEC §6.5 / §6.6).
- Allows composite embeddings that subsume multiple dimensions (SPEC §6.7).

The envelope is therefore a *structured, multi-source, multi-provenance, multi-facet object* — not a single vector.

---

## 3. The signal map of a v1 context envelope

What can ride in the envelope, by signal class, with typical providers and LSS assignments. This is illustrative; any provider can contribute to any dimension if their model is LSS-conformant.

### Media context (`med_emb`)

| Signal | Typical provider |
| --- | --- |
| Page content embedding | Publisher CMS, content-metadata vendor (e.g., Gracenote-class for CTV) |
| Show / episode embedding | CTV content metadata provider |
| App-screen / surface context | Publisher app instrumentation |
| Surrounding creative on the page | SSP, ad server |
| Content tone / brand-suitability | Brand-safety vendor (v2 separates this) |

### Physical context (`phy_emb`)

| Signal | Typical provider |
| --- | --- |
| Device class and form factor | Device platform, publisher |
| Motion state | Device SDK |
| Environment (in-home / out-of-home) | Device / publisher |
| H3 R8 place semantics | Geospatial provider (POI, mobility, census) |
| Connection class | Publisher / SSP |

### Temporal context (`tmp_emb`)

| Signal | Typical provider |
| --- | --- |
| Cyclical time encoding (hour / day / week / season) | ORP reference encoder (deterministic) |
| Recency within session | Publisher / SSP |
| Seasonality cycles | ORP reference encoder |
| Dwell within content | Publisher |
| Calendar-layer activations (religious, civic, academic, sports) | Calendar-layer provider |

### Social context (`soc_emb`)

| Signal | Typical provider |
| --- | --- |
| Co-presence proximity | Household graph provider (LiveRamp-class) |
| Household cohort signal | Household graph provider |
| Shared-device indication | Device platform, publisher |
| Public vs. private network | Publisher |

### Real-world context (`rwc_emb`)

| Signal | Typical provider |
| --- | --- |
| Sporting / live event | Sports data provider, real-time event API |
| Breaking news | News API |
| Weather observation | NOAA, weather API |
| Market move | Market data provider |
| Civic event | Open civic-data feeds |
| Permits / closures | City open-data portals |

---

## 4. The composite-embedding pattern

A provider may emit a **composite embedding** that subsumes signals from multiple dimensions into a single learned representation. This is structurally valid and often preferable when the provider has trained a model that captures cross-dimensional interactions better than separate per-dimension embeddings could.

The canonical worked example is Ether Data's **E_moment** — a place × time × world-state embedding that captures the joint state of a specific cell at a specific moment under specific live context. E_moment subsumes parts of `phy_emb`, `tmp_emb`, and `rwc_emb` into one learned representation while declaring its `composite_inputs` (place_h3r8, calendar_layers, weather_observation, school_session, transit_state, permits, poi_activity).

How a composite embedding appears in the envelope:

```json
{
  "type": "med_emb",
  "source_id": "etherdata.moment",
  "model_agreement_key": "ether-moment-v1-128",
  "dim": 128,
  "vector": "<base64...>",
  "runtime_caption": "Wednesday afternoon, Morningside Heights, 22°C clear. Columbia in move-in week. PS 36 in session, dismissal at 14:50. Block-party permit on 115th St. Restaurants, grocers, citibike active.",
  "regime_label": "context.university-move-in-spike",
  "provenance_uri": "https://etherdata.example/provenance/ether-moment-v1-128",
  "composite_inputs": [
    "place_h3r8",
    "calendar_layers",
    "weather_observation",
    "school_session",
    "transit_state",
    "permits",
    "poi_activity"
  ]
}
```

The same bid request can still carry elemental `phy_emb`, `tmp_emb`, `rwc_emb` from other providers. The ranker reads all of them and decides which to use; the composite is one signal among many, not a replacement for the elemental suite.

This is the "multiple providers contribute to one decision" pattern operationalized.

### When to emit a composite

A provider should emit a composite when:

- Their model captures cross-dimensional interactions that flat per-dimension embeddings would lose.
- The composite is the natural output of their existing production pipeline.
- The downstream rankers their customers run can ingest a composite directly.

A provider should emit elemental embeddings when:

- Their model only addresses one dimension.
- They want their signal to be usable by rankers that prefer the elemental suite.
- The dimensions they cover are sufficiently independent that no cross-dimensional value is being left on the table.

Many providers will emit both — a composite as their primary product, plus elemental projections of the same underlying state for consumers who want them.

---

## 5. Sparsity and activation

Many real-world context signals are *sparsely activated* — they fire only on a small fraction of impressions. Examples:

- **Ramadan calendar flag** — fires on ~30 days a year, on subsets of households.
- **School dismissal flag** — fires for a specific hour at a specific cell type.
- **Transit disruption flag** — fires for cells along affected lines during the affected window.
- **Major-event flag** — fires for cells near venues during event hours.

Every embedding in the envelope declares its `activation_rate` in the provenance manifest (SPEC §11.3). Sparsely-activated embeddings (`<0.1`) are evaluated under conditional signal density (SPEC §12.5 gate 2) — performance is measured on the subset where the signal fires, not the inactivated mass that would dilute the metric.

The envelope structure makes this honest: a transit-disruption signal that fires for 2% of impressions and adds large lift on those impressions is preserved as its own embedding, not folded into a denser context vector where its signal would be averaged away.

Sparsity is a first-class concept in ORP. The envelope is the carrier; the LSS framework is the evaluator; the runtime is the consumer.

---

## 6. Provenance and provider trust

Every embedding in the envelope arrives with full provenance. For context signals — which often touch sensitive dimensions (location, household, demographics, events) — the provenance burden is higher than for buyer-side embeddings.

What the receiving ranker can rely on for every context embedding:

- **`inference_class`** (SPEC §11.3) — declares whether the signal is observed directly (e.g., a sensor reading), inferred from proxies (e.g., religious composition from ACS ancestry data), or probabilistic (e.g., a learned cohort assignment). Captions must hedge appropriately for inferred / probabilistic signals (Priya's verdict in the Ether committee review).
- **`consent_purposes`** and **`jurisdictional_scope`** — per-embedding, not per-bid-request, because a context signal computed in jurisdiction A may be consumed in jurisdiction B with different consent semantics.
- **`activation_rate`** — what fraction of impressions this embedding fires for.
- **`conformance_attestations`** — what LSS gates this provider's model has passed.
- **`attestation_signature`** — cryptographic signature from the source operator over the provenance manifest.

The receiving ranker can — and in regulated verticals (healthcare, financial, alcohol, political, children) *must* — reject embeddings whose provenance does not satisfy the deployment's requirements.

---

## 7. The envelope at substrate scale

A continental-scale ORP deployment at H3 R8 grain produces large volumes. Some sizing intuition (from Ether Data's Lorenzo Madeleni in the E_moment committee review):

- US H3 R8 cells: ~300,000-500,000
- Cell × hour × year rows: 2.6-4.4 billion
- Cell × hour × signal × year rows: 10s of billions

Carrying full context envelopes in every bid request is not viable at this scale. ORP's pointer-based bidding pattern (SPEC §9.4) becomes operationally essential: the envelope can carry **embedding pointers** (`source_id` + `cell_id` + `model_agreement_key`) for durable, cacheable signals; the bidder resolves the pointers against its embedding-store cache.

What this looks like in practice:

- `phy_emb` for a given H3 cell is durable across many impressions in that cell. Emit it once, cache it at the bidder.
- `tmp_emb` for a given hour is durable across all impressions in that hour. Emit it once per hour-cell pair.
- `rwc_emb` for a given event window is durable across all impressions in the event's cell radius. Cache aggressively.
- `med_emb` is per-impression-content and less cacheable but still benefits from CDN-style caching by content URL.

The envelope thus carries a mix of **dense values** (small, frequently-changing signals) and **pointers** (large, cacheable signals). The bidder reconstitutes the full envelope at inference time.

This is the engineering pattern that makes the envelope deployable at national scale. Without it, the envelope concept is theoretically clean but operationally infeasible. With it, ORP scales to the volumes ad-tech actually operates at.

---

## 8. Bid-time flow for a context envelope

End-to-end illustration of how a context envelope assembles into a ranking call:

1. **Auction starts.** Publisher's SSP receives an impression request.
2. **SSP queries context providers.** Either via the SSP's pre-built integrations or via pointers to a shared embedding store, the SSP gathers context embeddings for this impression:
   - Publisher emits `med_emb` (page content).
   - Geospatial vendor emits `phy_emb` (place + device) for the H3 cell.
   - ORP reference encoder produces `tmp_emb` (cyclical time encoding).
   - Household-graph provider emits `soc_emb` if consent permits.
   - Real-time event provider emits `rwc_emb` if a salient event is firing.
   - Ether Data (or similar) emits the composite `med_emb` / E_moment, with `composite_inputs` declaring what it consumed.
3. **SSP assembles the envelope** into `ext.orp.embeddings`, with each embedding fully provenanced, captioned, regime-labeled.
4. **SSP fans out the bid request** to participating DSPs.
5. **Each DSP's ranker** reads the embeddings it knows how to consume, applies its proprietary scoring, returns a bid.
6. **Winning bid's caption set** travels back in the BidResponse extension as the decisioning trace ([docs/05-interpretability.md](05-interpretability.md)).

The envelope is the connective tissue between the multi-provider upstream and the multi-DSP downstream. ORP's standardization is what makes this fan-out / fan-in pattern viable.

---

## 9. Open questions specific to the envelope

These are documented as v1 open questions and will be resolved through the RFC process (SPEC §17).

- **Envelope-level vs. per-embedding consent.** When multiple consent regimes apply to different embeddings in the same envelope, what's the precedence? Currently per-embedding (SPEC §11) but operational complexity arguments for envelope-level rollup.
- **Forward-projected context.** Should the envelope expose a forward-projected variant (`context_envelope(cell, t+Δ)`) for planning use cases? Ether Data's Rafael argues yes for DOOH planning; Yuriy's serving-cost calculus may push back. Open.
- **Regime taxonomy depth.** The closed regime taxonomy must be hierarchical (SPEC §6.6); how deep is the right granularity? Too shallow and labels lose meaning; too deep and the taxonomy is unmaintainable. Probably 3 levels per dimension, but worth empirical validation.

---

## See also

- [docs/01-architecture.md](01-architecture.md) — the four phases, of which the context envelope is the Phase 1 deliverable
- [docs/02-latent-space.md](02-latent-space.md) — the LSS framework that defines what each envelope signal means
- [docs/04-provider-integration.md](04-provider-integration.md) — how a provider emits envelope-eligible embeddings
- [docs/05-interpretability.md](05-interpretability.md) — how the caption and regime fields make the envelope readable
- [docs/07-runtime-model.md](07-runtime-model.md) — end-to-end runtime flow showing the envelope in motion
- [SPEC.md §6.7](../SPEC.md#67-elemental-and-composite-embeddings) — elemental vs composite embeddings
- [SPEC.md §9](../SPEC.md#9-wire-format) — wire-format details
- [KB §19 Ether Data E_moment](../../knowledge-base/19-ether-emoment-proposal.md) — the canonical worked example for a composite context embedding
