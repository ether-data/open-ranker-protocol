# Runtime Model — What Happens at Bid Time

This document traces an ORP-conformant ad decision end to end: from the moment a publisher's impression is offered, through context gathering, through provider participation, through the ranker call, through bid response, through interpretability surfaces, through measurement loopback.

The intent is operational concreteness. Architecture, latent spaces, providers, interpretability, governance — these are abstract until you can answer the question: *what actually happens at bid time?*

---

## 1. Actors

Before tracing the flow, the actors involved:

| Actor | Role |
| --- | --- |
| **Publisher** | Owns the impression. Provides media context (`med_emb`) and the canonical input data the moment is encoded against |
| **SSP / exchange** | Receives the impression request from the publisher, assembles the ORP context envelope, fans out the bid request to DSPs |
| **Identity provider** | Emits `uid_emb` for the user (LiveRamp ATS, UID2 operator, OpenPass, first-party publisher graph) |
| **Context providers** | Emit `med_emb`, `phy_emb`, `tmp_emb`, `soc_emb`, `rwc_emb` — one or more per dimension, possibly including composite (E_moment-style) embeddings |
| **Embedding store** | Caches durable embeddings (`uid_emb`, `phy_emb` per cell, `tmp_emb` per hour) for pointer-based bidding |
| **Substrate operator** | Hosts the ranker and possibly the embedding-store cache; provides the inference environment (ARTF container, Index Cloud, AWS RTB Fabric) |
| **Bidder / DSP** | Runs the ranker. Reads embeddings, scores, returns a bid |
| **Creative / audience provider** | The bidder's own internal `cre_emb` and `aud_emb` services |
| **Reinforcement provider** | Emits `rnf_emb` if the user has consented and a feedback loop exists |
| **Measurement / attribution** | Receives the decisioning trace via loopback; closes the feedback loop |

Not every impression involves every actor. Many actors collapse — a publisher may also be the identity provider, a DSP may operate its own embedding store, a substrate operator may host the bidder.

---

## 2. The bid-time sequence

### Phase A — impression offered

1. **User opens content.** Publisher's surface (web page, app screen, CTV session) loads. An impression opportunity exists.
2. **Publisher signals SSP.** Standard ad-call to the SSP / exchange with the impression metadata (size, format, page URL, etc.). This part is OpenRTB, unchanged from today.

### Phase B — context envelope assembled

3. **SSP queries context providers.** For each context dimension the SSP supports, it gathers the relevant embedding:
   - **`med_emb`** — from the publisher's content metadata or a content-metadata provider.
   - **`phy_emb`** — from the device fingerprint + a geospatial provider (H3 cell ID, place semantics).
   - **`tmp_emb`** — deterministically computed by the ORP reference encoder (no external call needed for the cyclical encoding; calendar overlays may pull from a calendar provider).
   - **`soc_emb`** — from a household graph provider, if user consent permits.
   - **`rwc_emb`** — from a real-time event provider, if any salient event is firing for this cell × time.

4. **Composite contributions resolved.** If a composite-context provider is active (e.g., Ether Data emitting E_moment), the SSP pulls the composite embedding alongside the elemental suite. The composite declares its `composite_inputs`; the ranker can choose between the composite and the elementals.

5. **Pointer resolution where applicable.** For cached embeddings (durable `uid_emb`, per-cell `phy_emb`), the SSP emits pointers rather than full vectors. The bidder will resolve against its cache at inference time.

6. **Provenance and interpretability surfaces attached.** Every embedding instance carries its runtime caption, regime label, and provenance URI per SPEC §6.5–6.7.

The result: a complete `ext.orp.embeddings` array in the bid request, including the regime taxonomy URI at the top of the block.

### Phase C — identity binding

7. **Identity provider emits `uid_emb`.** Bound to the user's RampID / UID2 / OpenPass / publisher-first-party identifier, with consent posture declared per SPEC §11.
8. **Reinforcement signal attached if available.** `rnf_emb` from the measurement vendor or DSP feedback service, with the user's explicit consent.

### Phase D — bid request fanout

9. **SSP serializes the bid request.** The full OpenRTB bid request including `ext.orp` extension is sent to each participating DSP.
10. **DSPs receive in parallel.** Each DSP has milliseconds (typically 100-300 ms in the open web; 5-10 ms in ARTF-class container environments) to respond.

### Phase E — bidder processing

11. **Bidder parses the ORP extension.** Validates the wire format. Checks `model_agreement_key`s against its known LSS conformance set or its bilateral arrangements.
12. **Bidder resolves pointers** against its embedding-store cache for any pointer-emitted embeddings.
13. **Bidder applies fallback for unknown embeddings.** If an embedding has an unknown `model_agreement_key` and no translation is configured, the bidder either ignores that embedding or falls back to segment-based signals (SPEC §12.4).
14. **Bidder loads its own buyer-side embeddings.** `cre_emb` for the candidate creative; `aud_emb` for the campaign expression. Both are L2 — proprietary to the bidder.

### Phase F — ranking call

15. **The ranker call:**

```
score = ranker(
    uid_emb,        # who is here
    med_emb,        # what's being consumed
    phy_emb,        # where and how
    tmp_emb,        # when
    soc_emb,        # who's around
    rwc_emb,        # what's happening in the world
    cre_emb,        # the candidate creative
    aud_emb,        # what the campaign wants
    rnf_emb,        # how user has been responding
    [wst_emb],      # world-state prediction (v2, optional)
)
```

The ranker is the bidder's L2 IP. Common architectures: multi-tower DNN with separate input pathways per embedding, gradient boosting over flattened features, cross-encoder transformer, sequence-aware (HSTU-class) ranker for v2 deployments.

16. **Score informs bid value.** Bidder applies bid logic (target ROAS, frequency cap, pacing, etc.) to produce a final bid price.

### Phase G — bid response

17. **Bidder constructs BidResponse.** Standard OpenRTB fields plus optional `ext.orp.decisioning_trace` (SPEC §6 / [docs/05-interpretability.md §6](05-interpretability.md#6-the-decisioning-trace)):
   - Which embeddings the ranker actually consumed (by `source_id` + `model_agreement_key`).
   - The captions and regime labels of those embeddings.
   - Whether the ranker relied on LSS conformance, bilateral model agreement, or fallback.
   - Optional decisioning rationale (operator-published).

18. **SSP receives bids.** Auction logic determines the winning bid by price (+ quality factors per the SSP's policy).

### Phase H — ad serves and feedback loop

19. **Winning ad serves.** Standard creative delivery and measurement instrumentation.
20. **Outcome events fire.** Impression confirmed, view-through measured, click captured if it happens, conversion attributed if it happens.
21. **Decisioning trace persisted.** SSP retains the winning bid's `ext.orp.decisioning_trace` for audit. Publisher receives the trace as part of the impression record.
22. **Reinforcement signal updates.** Measurement vendor updates its `rnf_emb` for this user, incorporating the new outcome data. The updated embedding is available for future impressions.

The loop closes. The next impression for this user benefits from updated reinforcement signal.

---

## 3. A worked example — the Italy ad with full ORP plumbing

Tying every piece of the system to a single concrete impression. (This is the same example used in [SPEC.md Appendix A](../SPEC.md#appendix-a--worked-example-the-italy-ad), traced through the runtime here.)

### The scene

A user — call her A — spent three hours at a dance-studio gala that evening. Her friend at the same table has been searching for retirement villas in Italy. Later, on her couch, A opens Instagram on her phone.

### What happens in the bid request

| Phase | Actor | Action |
| --- | --- | --- |
| A | A's phone | Instagram-served impression opportunity in A's feed |
| B | Publisher (Instagram surface) | Sends ad call to SSP-equivalent |
| B | Content metadata provider | Emits `med_emb` for the page content (lifestyle feed, travel/vacation photos surrounding) |
| B | Geospatial provider | Emits `phy_emb` for A's H3 R8 cell (residential, indoors) plus device context (phone, stationary) |
| B | ORP reference encoder | Computes `tmp_emb` (Wednesday evening, dwell window, end-of-week-day cycle) |
| B | Household graph provider | Emits `soc_emb` with co-presence proximity earlier in the evening to A's friend (key signal — *no one was listening; the system understood*) |
| B | Real-time event provider | Returns no `rwc_emb` (no salient event at this cell × time) |
| C | LiveRamp ATS | Emits `uid_emb` for A (durable behavioral embedding: travel, design, family content) |
| C | Measurement vendor | Emits `rnf_emb` (mixed recent ad response; positive on Mediterranean food category) |
| D | SSP | Fans bid request to participating DSPs, including the one running the Italy travel campaign |

### What happens in the bidder

| Phase | Actor | Action |
| --- | --- | --- |
| E | DSP | Parses ORP extension, validates LSS conformance of each embedding |
| E | DSP | Loads its `cre_emb` for the candidate creative (Tuscan villa rental ad) |
| E | DSP | Loads its `aud_emb` for the campaign query ("U.S. adults with disposable income, in-market for Italian travel") — note the audience query is a *seed-audience-lookalike* type, and the seed includes users actively searching for Italian retirement villas |
| F | DSP ranker | Combines all inputs in its multi-input ranker. The signal that puts this impression over threshold is `soc_emb` — A's co-presence proximity earlier in the evening to a household-graph node who is in the seed audience. The bilinear interaction between `soc_emb` and `aud_emb` produces an unusually high score. |
| F | DSP | Applies bid logic; produces a bid price |
| G | DSP | Constructs BidResponse including decisioning trace: embeddings consumed (`uid_emb`, `soc_emb`, `tmp_emb`, `med_emb`, `phy_emb`, `cre_emb`, `aud_emb`), captions of each, regime labels of each |

### What A sees

The Tuscan villa ad serves in her feed.

### What the audit shows

Pulled from the BidResponse trace and the provenance manifests of the embeddings consumed:

- *Caption set on the winning bid:*
  - `uid_emb`: "Returning user with established travel and lifestyle engagement; medium recency."
  - `med_emb`: "Instagram feed, lifestyle / vacation content adjacent; evening dwell pattern."
  - `phy_emb`: "Phone, indoors, stationary; residential cell; evening lighting."
  - `tmp_emb`: "Wednesday evening; end-of-day dwell window."
  - `soc_emb`: "Adult household; co-presence proximity earlier today to household-graph node engaged with Italian-travel content (consent-based, ATS-aggregated)."
  - `cre_emb`: "Tuscan villa rental, professional photography, evocative copy."
  - `aud_emb`: "Campaign seeking adults with disposable income, in-market or proximate-to-in-market for Italian travel; expression_type=seed_audience_lookalike."

- *Regime labels:*
  - User: `user.engaged-lifestyle`
  - Media: `media.lifestyle-evening-discovery`
  - Physical: `physical.home-relaxed`
  - Temporal: `temporal.weekday-evening`
  - Social: `social.adult-household-cohort-proximity`
  - Audience: `audience.in-market-lookalike-travel`

- *Provenance assertions:*
  - All embeddings carry `consent_purposes` declarations and `jurisdictional_scope` consistent with A's TCF v2.2 string.
  - `soc_emb` from the household-graph provider declares `inference_class: probabilistic` (proximity inferred from cross-device household graph + temporal collocation); caption hedges appropriately.

The regulator-readable answer to "why did this ad serve to A?" is: *the system recognized that A was in proximity earlier that evening to a household-graph node actively in-market for Italian travel, and her own behavioral embedding showed travel and lifestyle engagement; the moment's context (evening, relaxed, mobile, lifestyle-feed adjacency) was favorable to a high-impact travel creative; the ranker combined those signals and bid above threshold.*

No one was listening. The system understood. And — critically — *the audit trail proves how*.

---

## 4. Performance characteristics

How long does this take?

| Phase | Open-web (legacy) | ARTF / containerized (ORP target) |
| --- | --- | --- |
| A: impression offered | <1 ms | <1 ms |
| B: context envelope assembled | 20-50 ms (sequential vendor calls + network) | 1-3 ms (co-located embedding store; substrate-cached) |
| C: identity binding | 5-15 ms | <1 ms (substrate-cached) |
| D: bid request fanout | 20-50 ms (network) | 1-2 ms (in-substrate routing) |
| E: bidder processing | 5-15 ms | 1-2 ms |
| F: ranking call | 5-30 ms (model inference) | 1-5 ms (on custom silicon: ~1 ms) |
| G: bid response | 20-50 ms (network) | 1-2 ms (in-substrate) |
| H: outcome and feedback | async | async |

Total round-trip:
- **Open-web baseline:** ~100-200 ms (limited by the 100 ms OpenRTB window most SSPs enforce).
- **ARTF / containerized target:** ~5-15 ms.

The 10-20× reduction comes from co-location (no inter-cloud network hops), substrate-cached embedding resolution (pointer-based bidding), and custom-silicon inference paths (Meta MTIA 2i benchmark: ~44% TCO reduction vs. commercial GPUs; see [KB §17](../../knowledge-base/17-meta-mtia2i-academic-paper.md)).

ORP is designed to operate efficiently in both regimes. The wire format is the same; the substrate determines the latency floor.

---

## 5. What this runtime model unlocks

End-state ORP-conformant ad decisioning provides:

1. **Multi-input ranking on the open web.** The same architectural pattern walled-garden rankers have used for a decade, with cross-vendor portability the walled gardens cannot offer.
2. **Multi-provider participation in one decision.** Many providers contribute signals; the ranker combines them. No single provider has to be the dominant one.
3. **Interpretability end-to-end.** Every embedding readable, every decision auditable, every consent declared and honored.
4. **Sub-10 ms decisioning at substrate scale.** When run on ARTF-class containerized substrates with embedding-store caching, the entire flow fits in a fraction of the OpenRTB window.
5. **Cookie-free / non-addressable yield.** Regime labels + context captions are sellable contextual signals even when no user identifier is present (see [Kai's verdict on KB §19](../../knowledge-base/19-ether-emoment-proposal.md#kai-digital-publisher-yield)).
6. **Regulator-defensible decisioning.** The interpretability surfaces (caption, regime, manifest, trace) answer the questions regulators have always asked, in a form that didn't exist in the embedding-only world.

This is the operational picture of the open web closing the substrate gap to the walled gardens.

---

## See also

- [docs/01-architecture.md](01-architecture.md) — the four phases this runtime model implements
- [docs/02-latent-space.md](02-latent-space.md) — LSS framework; the ranker reads embeddings against their LSSs
- [docs/03-context-envelope.md](03-context-envelope.md) — Phase B in detail
- [docs/04-provider-integration.md](04-provider-integration.md) — how each actor in this runtime gets there
- [docs/05-interpretability.md](05-interpretability.md) — the caption / regime / manifest / trace surfaces
- [docs/06-governance.md](06-governance.md) — why the runtime is structured this way
- [SPEC.md](../SPEC.md) — the wire format being exchanged at every step
- [SPEC.md Appendix A](../SPEC.md#appendix-a--worked-example-the-italy-ad) — the Italy ad example that this document expands to a full runtime trace
