# Interpretability

This document describes how ORP makes ad-decisioning *readable* — by advertisers, by publishers, by compliance reviewers, by regulators, and by the practitioners on each side who have to defend their use of the system to their organizations.

Interpretability is not an afterthought in ORP. It is a first-class spec requirement. Every embedding instance ships with a runtime caption and a regime label. Every provenance manifest declares model lineage and inference class. Every winning bid carries a decisioning trace. The system is designed to be readable end-to-end, not because that's nice, but because vectors that no one can read aren't deployable in regulated markets.

---

## 1. The interpretability gap embeddings create

The classical ad-decisioning audit trail looks like this:

> "We bid on this impression because the user was in the segment `IAB6-1: Cooking/Recipes` and the campaign targeted that segment."

The segment name is the answer. A compliance reviewer can read it. A brand legal team can defend it. A regulator can audit it.

The embedding-based audit trail by default looks like this:

> "We bid on this impression because the user-embedding had a cosine similarity of 0.83 with the audience-query embedding, exceeding the threshold of 0.78."

The numbers are the answer. No one can read them. No one can defend them. No one can audit them.

This is the problem ORP's interpretability surface exists to solve. Damian Naglak's note on this (filed in [KB §11](../../knowledge-base/11-segments-vs-embeddings.md)) was unambiguous: *"the audit trail is two arrays of numbers and a score, fine for an engineer, not useful for a compliance reviewer."* Without an answer to this, embeddings remain confined to corners of ad-tech where audit pressure is low — and the open web's parity story stalls.

---

## 2. The three interpretability surfaces

ORP solves the readability problem through three complementary surfaces, each addressing a different audience.

| Surface | Audience | When read | What it answers |
| --- | --- | --- | --- |
| **Runtime caption** | Compliance reviewers, brand legal, sales, regulators, ad-ops | At decisioning time and in post-hoc audit | What signal did the ranker see on this impression? |
| **Regime label** | Planners, optimization analysts, business intelligence, regulators | Continuously, queryable | What categorical state was this moment in? |
| **Provenance manifest** | Compliance, privacy officers, partner due-diligence, regulators | At onboarding and on demand | What does this embedding represent, where does it come from, what does it carry? |

Together these three answer the questions ad-tech audit has always had to answer — and that the embedding-only world had no way to.

---

## 3. Runtime captions

Every embedding instance in a bid request carries a `runtime_caption`: a deterministic, plain-language sentence rendered from the same canonical inputs the embedding consumed (SPEC §6.5).

### Why captions are faithful by construction

The caption is not generated *from* the embedding. It is generated *from the same input table* the encoder consumed. The encoder produces a vector; the caption template produces a sentence. Both consume the same canonical context (school flag, weather observation, transit state, calendar layer, POI activity). The caption is therefore faithful to what the encoder saw, by construction — not by approximation.

This is structurally important. An LLM-generated post-hoc explanation of a vector can hallucinate. A deterministic templated caption from the same input table cannot.

### How captions are produced

The caption template is a slot-filling structure published at `caption_template_uri` (SPEC §11.3). It is *not* a neural language model. It looks like this for a composite context embedding:

```
{daypart} {dow}, {neighborhood_name}, {weather_summary}.
{calendar_phrase}. {school_phrase}. {disruption_phrase}. {event_phrase}.
{poi_active_phrase}. {regime_phrase}.
```

Filled in for one moment:

> *"Wednesday afternoon, Morningside Heights, 22°C clear. No religious calendar active. Columbia in move-in week (3 days before fall semester); PS 36 in session, dismissal at 14:50. No transit disruption. Block-party permit on 115th St between Broadway and Amsterdam. Restaurants, grocers, citibike active. Regime: university-move-in-spike."*

The template is published; the slot values come from the canonical input table at the moment of encoding; the rendered sentence travels in the bid request alongside the vector.

### The hedging discipline

The provenance manifest declares `inference_class` per embedding (`measured`, `inferred`, `probabilistic`). Caption templates referencing inferred or probabilistic signals MUST hedge:

- ✅ *"Estimated high concentration of South Asian households (ACS ancestry proxy, ±15% uncertainty)."*
- ❌ *"Bay Ridge is 80% Bangladeshi-American."*

This rule comes directly from Priya's verdict on the Ether Data committee review (KB §19). Asserting probabilistic signals as flat facts in captions is a compliance failure mode. The hedging discipline prevents it.

### What captions enable

- **Audit by humans.** A compliance reviewer can read a winning bid's caption set and verify the system saw what it should have seen. No vector inspection required.
- **Sales conversations.** A buyer asking "what does the audience look like at the moment of activation" gets a sentence, not a vector. This is the commercial unlock Jason and Scout flagged in the Ether committee review.
- **Drift detection.** Captions on a sampled set of impressions, over time, reveal if the input table is drifting or if a signal is going stale. Visual drift detection without model archaeology.
- **Regulator response.** When a regulator asks why an ad served to a specific user in a specific moment, the caption set is the answer. Defensible because deterministic.

---

## 4. Regime labels

Every embedding instance also carries a `regime_label` — a categorical compression of the moment drawn from a closed, versioned taxonomy (SPEC §6.6).

### Why categorical alongside vector

A vector encodes nuance the planner can't query. A categorical label encodes the structural mode the impression is in, in a form planners and BI systems can filter on. The label is the bridge between embedding-based ranking and segment-based activation logic.

### The taxonomy

Regime labels are namespaced by dimension:

| Namespace | Example labels |
| --- | --- |
| `context.*` | `context.university-move-in-spike`, `context.transit-disrupted-weekend`, `context.holiday-iftar-window`, `context.major-sports-moment` |
| `physical.*` | `physical.home-relaxed`, `physical.commute-moving`, `physical.public-stationary`, `physical.outdoor-active` |
| `temporal.*` | `temporal.weekday-morning`, `temporal.weekday-afternoon`, `temporal.weekend-evening`, `temporal.holiday-eve` |
| `social.*` | `social.solo-adult`, `social.household-shared`, `social.public-multi-user` |
| `reinforcement.*` | `reinforcement.cold`, `reinforcement.selective-engaged`, `reinforcement.over-served`, `reinforcement.converted-recent` |
| `user.*` | `user.engaged-lifestyle`, `user.intent-active-research`, `user.passive-discovery`, `user.task-focused` |

The taxonomy is **closed** — labels are enumerated, not free text. It is **versioned** — each release has a stable URI referenced as `regime_taxonomy_uri` in the bid request. It is **hierarchical** — three levels deep typically (dimension / family / specific). It is **extensible** through the RFC process — labels are added in minor versions and not removed within a major version.

### What regime labels enable

- **Planner queryability.** "Show me all moments in `context.transit-disrupted-weekend` across my locations" is now a database query, not an LLM prompt.
- **BI / dashboard surfaces.** Regime distributions over time become charts; over geographies become maps.
- **Compliance scoping.** A brand-safety policy that excludes `physical.public-stationary` during regulated-product campaigns is enforceable as a regime filter at the SSP or DSP level.
- **Insurance / actuarial audit.** Regulated buyers (insurance, finance, pharma) get a discrete, defensible decisioning trace.
- **Cross-vendor consistency.** Two providers emitting `med_emb` to the same taxonomy version use the same regime label space. Aggregation across providers is meaningful.

### What the taxonomy is not

- Not a free-form tag system. Adding labels requires RFC, not API call.
- Not a substitute for the IAB Audience Taxonomy. The IAB taxonomy describes *audiences*; the regime taxonomy describes *moments and states*. They coexist.
- Not a black-box label assignment. The provider declaring a label commits to its definitional criteria from the taxonomy spec; auditors can verify.

---

## 5. The provenance manifest

The third interpretability surface is the provenance manifest (SPEC §11.3). Where the caption answers "what did the system see *this time*" and the regime label answers "what *state* was it in," the manifest answers "what does this embedding *mean* in general, where does it come from, and what does it carry."

The manifest is consumed at onboarding (a partner integrating a new provider's embeddings) and on-demand (a regulator or auditor doing diligence). It is the deepest of the three surfaces and the one that most rarely gets read — but when it gets read, it has to answer everything.

Key fields for interpretability:

| Field | What it tells the reader |
| --- | --- |
| `embedding_type` and `embedding_class` | What space this embedding is in; whether it's elemental or composite |
| `composite_inputs` | If composite, what input pipelines were combined |
| `model_lineage` | What model architecture; what training data; what training objective |
| `inference_class` | Whether signals are measured, inferred, or probabilistic — drives caption hedging |
| `data_provenance` | What populations the embedding is derived from |
| `population_excluded` | Populations explicitly excluded (children, sensitive categories) |
| `consent_purposes` and `jurisdictional_scope` | What consents the embedding carries and where they apply |
| `activation_rate` | How sparse the embedding is — drives ranker null-handling logic |
| `caption_template_uri` | Where to verify caption generation is deterministic |
| `regime_taxonomy_uri` | What taxonomy version regime labels are drawn from |
| `conformance_attestations` | What LSS gates the model has passed |
| `update_cadence` | How often the embedding is recomputed |
| `attestation_signature` | Cryptographic signature from the provider over the manifest |

A receiving consumer can — and in regulated verticals must — reject embeddings whose manifest does not satisfy the deployment's requirements.

---

## 6. The decisioning trace

When an ORP-conformant ranker wins a bid, the BidResponse extension carries a **decisioning trace** that records what the ranker saw and how it combined inputs. This is the audit artifact for a specific winning impression.

The trace includes:

- **Embeddings consumed.** Which embedding instances the ranker actually used (by `source_id` + `model_agreement_key`).
- **Captions consumed.** The runtime captions of those embeddings.
- **Regime labels consumed.** The regime labels of those embeddings.
- **Conformance reliance.** Whether the ranker relied on LSS conformance, bilateral model agreement, or fallback to non-embedding signals.
- **Decisioning rationale (optional, operator-published).** Why the ranker chose this bid value. The form is operator-defined but must be human-readable.

This is what travels back to the publisher and (eventually, through the measurement loopback) to the advertiser. It is the closing of the audit loop that the embedding-only world never had.

The decisioning trace is optional from the bidder's perspective in v1 — operators can withhold for competitive reasons. v2 may make it mandatory; the open question is how to mandate without compromising operator-proprietary ranker IP. RFC discussion is open.

---

## 7. How advertisers read the system

An advertiser's questions and how ORP answers them:

| Advertiser question | What surface answers it |
| --- | --- |
| Why did my ad serve here? | Runtime caption set on the winning bid |
| What kind of moment was this? | Regime label set on the winning bid |
| What signal did the ranker rely on? | Decisioning trace |
| Who produced the embeddings the ranker consumed? | `source_id` per embedding in the trace |
| Were those embeddings conformant? | Conformance attestations in each provider's manifest |
| What populations were excluded from training? | `population_excluded` in each manifest |
| What consents did this rely on? | `consent_purposes` + `jurisdictional_scope` per embedding |
| Is the inference probabilistic or measured? | `inference_class` per embedding |
| How often does the embedding refresh? | `update_cadence` per embedding |

All of these answer through standard fields on standard surfaces. The advertiser doesn't need to read vectors; they read sentences and structured manifests.

---

## 8. How publishers read the system

A publisher's questions are different from an advertiser's but ORP serves both:

| Publisher question | What surface answers it |
| --- | --- |
| What kind of moments is my inventory being matched against? | Regime label distribution over time |
| What signals are downstream rankers relying on? | Decisioning traces from winning bids |
| Are providers respecting my consent posture? | `consent_purposes` per embedding the ranker consumed |
| Is my non-addressable inventory monetizable in the embedding world? | Yes — see [Kai's verdict on KB §19](../../knowledge-base/19-ether-emoment-proposal.md#kai-digital-publisher-yield); regime + caption surfaces drive yield without requiring user identifiers |
| What new signals could I emit to add yield? | Provenance manifests of competing context providers reveal what's in market |
| Am I being shaped against unfairly? | Decisioning trace shows which signals the ranker actually used |

The publisher reads the same surfaces as the advertiser but from a different perspective. ORP is one standard, two readerships.

---

## 9. How compliance and regulators read the system

The regulatory perspective is the strictest reader of the system and the one that justifies the entire interpretability discipline.

| Regulator question | What surface answers it |
| --- | --- |
| What does this embedding represent and where does it come from? | Provenance manifest |
| Is the underlying signal measured or inferred? | `inference_class` |
| Are children or sensitive populations excluded? | `population_excluded` |
| What consents apply and in what jurisdictions? | `consent_purposes` + `jurisdictional_scope` |
| Was this user's right-to-erasure honored? | Erasure-endpoint reference in the manifest (SPEC §11.2) |
| Why did this ad serve to this user at this moment? | Runtime caption set on the winning bid |
| What categorical state was the moment in? | Regime label |
| Has the model passed published fairness gates? | Conformance attestations (especially spatial-equity gate, §12.5) |
| Can the audit be reproduced? | LSS reference evaluation set is published; provider's attestations can be re-run |

This is the test ORP's interpretability framework has to pass. The answer to each row above is a standard field on a standard surface, machine-readable and human-readable. The framework is designed against this regulator-readability bar.

---

## 10. The interpretability discipline in v1

What's mandatory in v1:

- Runtime caption per embedding instance.
- Regime label per embedding instance, from a published closed taxonomy.
- Provenance manifest per embedding type, with all SPEC §11.3 fields populated.
- Caption hedging for inferred / probabilistic inference classes.
- Conformance attestations per LSS-claimed embedding.

What's optional in v1, planned for v2:

- Full decisioning trace in BidResponse extension. v1 makes this optional to avoid forcing operators to disclose ranker IP before community consensus.
- Third-party attestation of provenance manifests (independent auditors). v1 supports self-attestation; v2 may require third-party for certain regulated verticals.
- Standardized fairness / equity dashboards. Spatial-equity is in the gate framework; broader fairness reporting is v2.

The v1 surface is *enough* to make ORP deployable in regulated verticals. v2 strengthens what's already there rather than adding new surfaces.

---

## See also

- [SPEC.md §6.5–6.6](../SPEC.md#65-every-embedding-instance-ships-with-a-runtime-caption) — runtime captions and regime labels
- [SPEC.md §11.3](../SPEC.md#113-provenance-manifest) — provenance manifest
- [SPEC.md §12.5](../SPEC.md#125-conformance-gates--what-an-lss-conformant-embedding-must-clear) — conformance gates
- [docs/02-latent-space.md](02-latent-space.md) — LSS framework that provenance manifests attest against
- [docs/04-provider-integration.md](04-provider-integration.md) — how providers publish the interpretability surfaces
- [KB §11 segments vs embeddings](../../knowledge-base/11-segments-vs-embeddings.md) — Damian's original framing of the audit-trail gap
- [KB §19 Ether Data E_moment](../../knowledge-base/19-ether-emoment-proposal.md) — caption decoder design and the descriptivity discipline
