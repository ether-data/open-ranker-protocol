# Open Ranker Protocol (ORP)

## A multi-dimensional embedding exchange specification for the open-web ad stack

**Version:** 0.1 (draft, open for comment)
**Status:** Proposed — invite to community
**Author:** Nathan Woodman
**License:** Specification text under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). Reference implementations under [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

> The name *Open Ranker Protocol* is a placeholder for community discussion. The spec may be renamed before v1.0.

---

## Table of contents

1. [Abstract](#1-abstract)
2. [Status](#2-status-of-this-document)
3. [Motivation](#3-motivation)
4. [Background: how walled-garden rankers work](#4-background-how-walled-garden-rankers-work)
5. [Goals and non-goals](#5-goals-and-non-goals)
6. [Architecture overview](#6-architecture-overview)
7. [The embedding catalog — v1](#7-the-embedding-catalog--v1)
8. [Extension embeddings — v2 roadmap](#8-extension-embeddings--v2-roadmap)
9. [Wire format](#9-wire-format)
10. [Identity binding](#10-identity-binding)
11. [Privacy, consent, and audit](#11-privacy-consent-and-audit)
12. [Model interoperability](#12-model-interoperability)
13. [Versioning](#13-versioning)
14. [Reference implementation](#14-reference-implementation)
15. [Security considerations](#15-security-considerations)
16. [Relationship to existing specs](#16-relationship-to-existing-specs)
17. [Open questions](#17-open-questions)
18. [Governance](#18-governance)
19. [Glossary](#19-glossary)

---

## 1. Abstract

The Open Ranker Protocol (ORP) is a proposed open standard for exchanging the embedding inputs used by modern ad-ranking systems across the open web. It treats the user, the creative, the audience query, the reinforcement signal, and — critically — **each distinct facet of context** as separate first-class embedding spaces that travel together through the bid request and feed a multi-input ranking model at decision time.

Context, in particular, is not a single dimension. The media being consumed, the physical setting, the time, the social proximity, and the unfolding real world are each their own input to a serious ranker. Walled gardens — Meta, YouTube, TikTok, Amazon — have always treated context this way internally. ORP makes that architecture portable, vendor-neutral, and interoperable so the open web can compete on the same primitives.

ORP extends the foundation laid by LiveRamp's User Context Protocol (donated to IAB Tech Lab as Agentic Audiences) by expanding context into a multi-embedding suite, adding creative and audience-query as first-class embedding spaces, specifying wire formats, addressing the model-interoperability problem head-on, and coexisting with segment-based targeting (IAB Audience Taxonomy).

## 2. Status of this document

This is a v0.1 draft published for community comment. The spec is intended to be developed in the open under permissive licenses, with the long-term goal of contribution to a neutral standards body (e.g., IAB Tech Lab as part of the AAMP framework, or W3C, depending on community preference).

Comments, pull requests, and counter-proposals are invited. See [§18 Governance](#18-governance).

## 3. Motivation

Today's open-web ad stack relies on three categories of audience representation:

- **Identity** — RampID, UID2, hashed emails, OpenPass.
- **Audience taxonomies** — IAB Audience Taxonomy v1.1, provider-defined segments.
- **Contextual signals** — page-level, content-level, app-level metadata.

What the open web does not yet have is a shared protocol for **learned representations** — the embeddings that walled-garden rankers use to combine many dimensions of signal in a single ranking call.

LiveRamp's User Context Protocol (UCP, now [Agentic Audiences](https://github.com/IABTechLab/agentic-audiences)) was the first move toward this. It defines 256–1,024 dimensional embeddings for three signal types: identity, contextual, and reinforcement. Reference implementations include ATS.js + Prebid for browser-edge embedding storage and a Docker sidecar for campaign scoring.

ORP builds on Agentic Audiences by:

- Treating **creative** and **audience-query** as first-class embedding spaces alongside identity, context, and reinforcement.
- Specifying how these embeddings combine in a multi-input ranking call.
- Addressing **model interoperability** as a first-class problem rather than an implementation detail.
- Defining wire formats for the bid request.
- Specifying identity binding to existing identifiers.
- Mandating a **human-readable provenance manifest** alongside each embedding to solve the audit-trail gap.
- Coexisting with segment-based targeting — the bid request carries both.

The performance gap between the open web and the walled gardens is not primarily a data gap. It is a substrate gap. Walled gardens combine many dimensions of context inside a learned ranking model at decision time; the open web has historically combined a few dimensions inside Boolean targeting statements outside the ranking model. ORP is one piece of closing that gap.

## 4. Background: how walled-garden rankers work

Modern recommendation and advertising systems at consumer scale do not match on a single representation of who the user is. They run a learned **ranking model** that takes multiple distinct embedding inputs and produces a ranking score per candidate item.

The canonical pattern is a multi-input architecture (often referred to as multi-tower):

```
user_emb ──┐
context_emb ──┤
                                ranker(...) → score
creative_emb ──┤
audience_query_emb ──┤
reinforcement_emb ──┘
```

Each embedding is learned end-to-end with respect to the ranking objective (clicks, conversions, watch time, engagement). The user-side embeddings are typically learned from behavioral sequences, not from text descriptions. Item/content/creative embeddings often have multimodal inputs (text, image, video) but their position in the embedding space is shaped by interaction supervision, not by lexical similarity.

References for the curious:
- Covington, Adams, Sargin (2016) — "Deep Neural Networks for YouTube Recommendations"
- Naumov et al. (2019) — "Deep Learning Recommendation Model for Personalization and Recommendation Systems" (Meta DLRM)
- Ying et al. (2018) — "Graph Convolutional Neural Networks for Web-Scale Recommender Systems" (Pinterest PinSage)
- Liu et al. (2022) — "Monolith: Real Time Recommendation System With Collisionless Embedding Table" (ByteDance)

The open web has not had access to this architecture not because the technique is unknown, but because the open web has not had:

1. A place to run inference at bid-time with sub-10ms latency and the full ranking model loaded (now addressed by [ARTF](https://iabtechlab.com/standards/artf/) / containerized RTB).
2. A protocol for moving the embedding inputs across vendor boundaries (partially addressed by Agentic Audiences; further addressed by ORP).
3. A practical framework for cross-vendor model interoperability (addressed by ORP §12).

## 5. Goals and non-goals

### 5.1 Goals

- Define a wire format that lets bidders, ad servers, and ranking services exchange the embeddings necessary for a multi-input ranking call inside the bid request.
- Treat each embedding space as first-class, with its own schema, model lineage, provenance manifest, and privacy controls.
- Coexist with segment-based targeting (IAB Audience Taxonomy and provider segments). The bid request carries both; buyers choose per use case.
- Enable cross-vendor portability of learned representations.
- Be implementable inside containerized RTB environments ([ARTF](https://iabtechlab.com/standards/artf/), Index Cloud, AWS RTB Fabric) and standard cloud-mesh environments.
- Provide a human-readable provenance manifest with each embedding so compliance reviewers, regulators, and brand legal teams can audit what an embedding represents.
- Define an explicit model-agreement protocol so two systems can know whether their embeddings are comparable before they try to compare them.

### 5.2 Non-goals

- **Replacing segment-based targeting.** Segments and embeddings serve different use cases (clean set logic, regulatory auditability, atomic dedup vs. continuous ranking, natural-language audiences, nuance). See [§16](#16-relationship-to-existing-specs).
- **Mandating any specific model architecture.** ORP defines schemas, wire formats, and interoperability negotiation. Vendors choose model architectures (two-tower, cross-encoder, transformer-on-events, graph-based).
- **Replacing identity middleware.** ORP binds to existing identifiers (RampID, UID2, OpenPass, hashed emails, publisher-first-party IDs) — it does not replace them.
- **Defining auction logic.** ORP defines inputs to a ranker. Auction logic, floor pricing, and decisioning remain with the host platform.
- **Replacing OpenRTB.** ORP adds optional fields to OpenRTB bid requests; OpenRTB remains the transport.
- **Solving content-moderation, brand-safety, or fraud detection.** ORP is composable with brand-safety and fraud-detection vendors but does not specify their behavior.

## 6. Architecture overview

### 6.1 The ranker as a function of multiple embeddings

A ranking call under ORP is:

```
score = ranker(
    user_identity_emb,
    context_emb,
    creative_emb,
    audience_query_emb,
    reinforcement_emb,
    [extension embeddings...],
    [non-embedding features: segments, signals, etc.]
)
```

The ranker is owned by the buyer (or the buyer's containerized inference service running inside the host environment via ARTF). ORP does not specify the ranker. ORP specifies the inputs.

### 6.2 First-class embeddings (v1)

**Context is not a single dimension.** A serious ranking call needs context broken out as a *suite* of distinct embedding inputs, each describing a different facet of the moment. The walled gardens have always done this internally; collapsing them into one "context embedding" would lose the very signal the spec is trying to make portable.

The v1 specification defines nine first-class embedding spaces across three conceptual groupings.

**User-side embeddings — who is here and how they have responded:**

| Embedding | What it represents | Typical source |
| --- | --- | --- |
| **User Identity** (`uid_emb`) | Who this person is, learned from durable behavioral history | Identity middleware (LiveRamp ATS, UID2 operators, first-party publisher, retailer) |
| **Reinforcement** (`rnf_emb`) | How this user has responded to recent advertising | Measurement provider, DSP feedback, retailer post-purchase signal |

**Context suite — five distinct facets of the moment:**

| Embedding | What it represents | Typical source |
| --- | --- | --- |
| **Media Context** (`med_emb`) | What is being consumed — page, show, app screen, podcast episode, surrounding creative | Publisher, content-metadata provider (Gracenote-style for CTV), SSP |
| **Physical Context** (`phy_emb`) | Where and how — device, form factor, motion, environment, location semantics | Device platform, publisher, geo-signal provider |
| **Temporal Context** (`tmp_emb`) | When — time of day, day of week, recency, seasonality, dwell | Encoded at impression time |
| **Social Context** (`soc_emb`) | Who is around — co-presence proximity, household/cohort signal, surrounding social cues | Identity provider, household graph, device graph |
| **Real-World Context** (`rwc_emb`) | What is happening in the world right now — events unfolding, weather, news, market moves | Real-time signal aggregator, news provider, weather/event API |

**Buyer-side embeddings — what the campaign wants and what is on offer:**

| Embedding | What it represents | Typical source |
| --- | --- | --- |
| **Creative** (`cre_emb`) | The ad itself — copy, image, video, format, brand attributes | Creative agency, DSP, ad server |
| **Audience Query** (`aud_emb`) | What the campaign wants in this moment | DSP, custom-algo vendor (Chalice / Scibids / Empowered), agency planning tool |

Each embedding has a v1 minimum schema (§7) and travels with a provenance manifest (§11.3). Not every embedding will be present in every bid request — implementations declare what they emit and what they consume, and the ranker handles missing inputs.

### 6.3 Extension embeddings (v2 roadmap)

v2 adds these as optional first-class spaces:

- **Placement** (`plc_emb`) — the slot itself: format, viewability, attention, position-on-page
- **Brand-safety** (`bsf_emb`) — brand-context fit, suitability, GARM categories
- **Inventory / Supply** (`inv_emb`) — supply-chain attributes, sellers.json signal, supply-path priors
- **Outcome** (`out_emb`) — durable post-conversion patterns feeding back into `rnf_emb` updates

The v1/v2 split is deliberate. v1 covers the nine inputs a multi-dimensional bid-time ranker actually needs to act on the moment. v2 expands toward placement-aware, supply-chain-aware, and longer-horizon outcome modeling.

### 6.4 Coexistence with segment-based targeting

The bid request continues to carry IAB Audience Taxonomy segment IDs and provider segments alongside ORP embeddings. Buyers choose per use case. Some matches are best done by ID intersection (clean set logic, regulatory audit, atomic dedup); some are best done by embedding similarity (natural-language audiences, ranking, nuance). ORP does not dictate which.

This coexistence is non-negotiable. It is the only way the spec can be adopted incrementally without breaking working inventory.

### 6.5 Every embedding instance ships with a runtime caption

Each first-class embedding instance in a bid request MUST be accompanied by a **runtime caption** — a deterministic, templated, plain-language description rendered from the same canonical inputs that produced the embedding. The caption is faithful by construction because both the encoder and the caption decoder consume the same input table.

The runtime caption serves three purposes:

1. **Audit.** Compliance reviewers, brand legal teams, and regulators can read the caption to verify what the embedding represents without inspecting vectors.
2. **Interpretability.** Both advertisers and publishers can read the same caption to understand what signal the ranker saw.
3. **Decisioning trace.** A winning bid's caption set becomes the explainability artifact for that impression.

Caption generation is non-neural. It is a slot-filling template driven by the structured canonical inputs feeding the encoder. The template is referenced via `caption_template_uri` in the provenance manifest (§11.3); the rendered instance travels with the embedding in the bid request.

### 6.6 Every embedding instance carries a regime label from a closed taxonomy

Each first-class embedding instance MUST also carry a `regime_label` — a categorical compression of the embedded moment drawn from a published, versioned, closed taxonomy.

The regime label is *queryable* in ways the vector is not. Planners can filter on "show me all impressions in regime X." Compliance reviewers can audit at regime granularity. The IAB Audience Taxonomy / segment-style discipline carries over to the embedding world.

Critically: the regime taxonomy is closed and versioned. It is never a free-text field. The taxonomy itself is governed under §18 and extended through the RFC process. Each embedding instance declares the `regime_taxonomy_uri` it draws from and the specific label assigned for this moment.

### 6.7 Elemental and composite embeddings

ORP recognizes two structural classes of first-class embeddings:

- **Elemental embedding** — produced by a single encoder over a coherent input pipeline (e.g., a `med_emb` from a content-metadata provider).
- **Composite embedding** — produced by a structured product of multiple input pipelines, often spanning multiple ORP dimensions (e.g., a place × time × world-state embedding that subsumes parts of `phy_emb`, `tmp_emb`, and `rwc_emb` into a single learned representation).

Both are valid. The provenance manifest (§11.3) declares which class the embedding belongs to via `composite_inputs` — an enumerated list of the canonical input pipelines the embedding consumed. Consumers (ranker operators) choose between consuming a composite embedding or its elemental components depending on what's available and what their ranker is built to ingest.

This recognition is important: real production embedding pipelines often combine multiple ORP dimensions into a single learned representation for compute efficiency or for capturing cross-dimensional interactions that flat suites would miss.

## 7. The embedding catalog — v1

Each first-class embedding is defined by:

- **Semantic definition** — what the embedding represents.
- **Schema** — dimensionality, dtype, normalization, encoding.
- **Provenance manifest** — the human- and machine-readable description that travels with the embedding (§11.3).
- **Identity binding** — which identifier the embedding is keyed to (§10).
- **Privacy model** — consent, TTL, jurisdiction (§11).

All v1 embeddings share a common base schema:

- Dimensions: 256, 384, 512, 768, or 1,024 (must declare)
- dtype: float16 or float32 (float16 RECOMMENDED for wire efficiency)
- Normalization: L2-normalized (unit length) REQUIRED
- Encoding: base64 over compact binary, OR JSON array of floats with `compressed: true` flag
- Provenance: REQUIRED (§11.3)
- Model agreement key: REQUIRED (§12.2)

Type-specific schema additions are noted per embedding below.

### 7.1 User-side embeddings

#### 7.1.1 User Identity Embedding (`uid_emb`)

**Semantic definition:** A dense vector representation of the user, learned from durable behavioral history. Captures interests, intent patterns, content affinities, and engagement style. Is *not* a literal identifier — it is a representation.

**Source candidates:** LiveRamp ATS, UID2 operator, first-party publisher graph, retailer customer graph, agency-side DMP. Multiple `uid_emb` instances MAY appear in a single bid request from different sources.

**Notes:**
- `uid_emb` is durable and benefits from pointer-based emission (§9.4).

#### 7.1.2 Reinforcement Embedding (`rnf_emb`)

**Semantic definition:** A dense vector encoding how this user has responded to recent advertising — impressions, clicks, conversions, dwell time, view-through, post-impression search behavior. The user side of the closed loop.

**Source candidates:** Measurement provider, DSP feedback service, retailer post-purchase signal, publisher engagement signal.

**Notes:**
- `rnf_emb` is the most privacy-sensitive embedding in v1 and MUST carry a consent assertion in its provenance manifest (§11).
- Implementations SHOULD support a "reinforcement-omitted" mode where the buyer can rank without `rnf_emb` if the user has not consented to feedback aggregation.

### 7.2 Context-suite embeddings

The context suite is the part of ORP that explicitly rejects collapsing "context" into a single representation. Each of the five context dimensions below describes a different facet of the *moment* in which the impression occurs. Each is its own input to the ranker. Each may come from a different source. None of them is a representation of the user.

A bid request MAY carry any subset of the five. The ranker handles missing inputs.

#### 7.2.1 Media Context Embedding (`med_emb`)

**Semantic definition:** A dense vector representation of what is being consumed in the moment: page content, show/episode, app screen, podcast episode, surrounding creative, content adjacency.

**Source candidates:** Publisher, content-metadata provider (Gracenote, Comscore), CMS, SSP. Multiple `med_emb` instances MAY appear (e.g., one from the publisher's CMS, one from a third-party metadata provider).

**Notes:**
- This is the "content embedding" half of the multi-tower architecture in most consumer recommendation systems.
- For CTV, the show/episode-level embedding lives here.
- For commerce contexts, the product-listing-context embedding lives here.

#### 7.2.2 Physical Context Embedding (`phy_emb`)

**Semantic definition:** A dense vector representation of *where* and *how* the impression is occurring: device type and form factor, motion state, environment (in-home/out-of-home, near-field/far-field for audio), location semantics (a place, not just coordinates), connection class.

**Source candidates:** Device platform, publisher, geo-signal provider, mobility-data provider.

**Spatial identifier (RECOMMENDED):**
- `cell_system: "h3"` and `cell_resolution: 8` is the RECOMMENDED default for the spatial identifier carried alongside `phy_emb`. H3 R8 (~460 m hexagonal edge) is the grain at which place-scoped state can be meaningfully modeled and at which national-scale rollouts remain materializable (~300-500k cells in the US).
- Other cell systems MAY be declared via `cell_system_id` (e.g., `"geohash"`, `"s2"`, `"plus_codes"`, custom). Receivers SHOULD be able to translate between H3, S2, and Geohash for interop.
- The cell identifier itself is carried in `provenance.spatial_binding.cell_id`. Raw lat/long MUST NOT be carried in the bid request as a substitute for the cell identifier.

**Notes:**
- A 1-bedroom kitchen at 7am and a stadium at 9pm both have GPS coordinates. The `phy_emb` is what makes them meaningfully different to a ranker.
- Raw lat/long should NOT be used as a substitute. The `phy_emb` is a *learned semantic representation* of the place and the situation.
- The cell identifier is metadata, not the embedding itself. The embedding captures the *learned* representation of place + device + situation; the cell identifier provides the canonical spatial address for joining external data.

#### 7.2.3 Temporal Context Embedding (`tmp_emb`)

**Semantic definition:** A dense vector representation of *when* the impression occurs in a way the ranker can use directly: time-of-day, day-of-week, recency relative to recent activity, seasonality cycles, dwell within the current session.

**Source candidates:** Encoded at impression time by the publisher or SSP; standardizable so a single open implementation can serve the whole ecosystem.

**Notes:**
- This is one of the lowest-bandwidth, highest-ROI embeddings in the suite. Production rankers consistently find temporal features among their top signals.
- A reference v1 implementation will publish an open, deterministic temporal embedder so every operator can produce comparable `tmp_emb` instances without licensing a model.

#### 7.2.4 Social Context Embedding (`soc_emb`)

**Semantic definition:** A dense vector representation of *who is around* and *what social setting* the impression sits in: co-presence proximity, household / cohort signal, surrounding social cues that the system can observe (a shared device, a multi-user CTV session, a public-network connection).

**Source candidates:** Identity provider, household graph, device graph, retailer co-purchase graph.

**Notes:**
- `soc_emb` is the most novel embedding in the v1 catalog and the most powerful for the kinds of multi-dimensional inference walled gardens excel at. It is what lets the system understand that two people at the same table for three hours probably share intent without anyone listening to them.
- `soc_emb` carries elevated privacy obligations. Implementations MUST honor jurisdiction-specific household-signal rules.

#### 7.2.5 Real-World Context Embedding (`rwc_emb`)

**Semantic definition:** A dense vector representation of what is happening in the world *around* the impression: events unfolding (a sports moment, a breaking-news event, a market move), weather conditions, civic events, cultural moments.

**Source candidates:** Real-time signal aggregator, news API, weather/event provider, sports data provider, market data provider.

**Notes:**
- `rwc_emb` is the embedding most likely to be impression-specific and non-cacheable.
- For sports/live event monetization (where containerized RTB has already proven its value), `rwc_emb` is what lets a ranker know the moment is a commercial break after a game-winning play vs. a routine break in a 35-7 blowout.

### 7.3 Buyer-side embeddings

#### 7.3.1 Creative Embedding (`cre_emb`)

**Semantic definition:** A dense vector representation of the candidate ad creative. Captures copy semantics, image/video content, format, brand attributes, and creative-context fit signals.

**Source candidates:** Creative agency, DSP creative service, ad server, brand's in-house creative tool.

**Type-specific schema additions:**
- `creative_id` — REQUIRED, ties embedding to the underlying creative
- Multimodal flag — declare which modalities (text, image, video, audio) are fused in the embedding, in the provenance manifest

**Notes:**
- `cre_emb` lives on the buyer side. It is not transmitted in the bid request from the SSP. It is loaded by the bidder when scoring candidate creatives.
- Including it in this spec is intentional: vendors need to publish creative-embedding schemas so a ranker can combine them with the user-side and context-side embeddings consistently.

#### 7.3.2 Audience Query Embedding (`aud_emb`)

**Semantic definition:** A dense vector representation of *what the campaign wants in this moment*. The buyer-side description of the audience the campaign is trying to reach, encoded as a vector in a space the ranker can compare to the user-side embeddings via §12.

**Source candidates:** DSP, custom-bidding-algo vendor (Chalice, Scibids, Empowered), agency planning tool, brand's in-house seat.

**Type-specific schema additions:**
- `query_id` — REQUIRED, ties embedding to the query expression
- `expression_type` — `natural_language`, `seed_audience_lookalike`, `behavioral_query`, `composite` (REQUIRED)
- `expression_artifact` — the human-readable description, seed-audience-ID, or query DSL that produced the embedding (REQUIRED in provenance manifest)

**Notes:**
- The `aud_emb` is what makes ORP a *ranker* protocol rather than just an audience protocol. The buyer brings *what they want*; the user side brings *who is here*; the context suite brings *what kind of moment this is*; the ranker decides whether to bid and at what price.
- The expression_artifact is critical for audit. "Premium auto intenders in the U.S. likely to be in-market in the next 30 days" is what travels alongside the embedding for compliance review (§11.3).

### 7.4 Multiple instances per embedding type

A single bid request MAY carry multiple instances of the same embedding type from different sources (e.g., two `med_emb` from different metadata providers, two `uid_emb` from different identity providers, two `phy_emb` from device and geo-signal sources). Each instance is independently scoped to its model agreement key and provenance manifest. The ranker chooses which to use, and SHOULD log its choice in the BidResponse extension for audit.

## 8. Extension embeddings — v2 roadmap

The v2 spec MAY add these as optional first-class embeddings. They are listed here so implementers can leave room.

- **Placement Embedding (`plc_emb`)** — slot characteristics: format, viewability priors, attention priors, position-on-page, scroll behavior.
- **Brand-Safety Embedding (`bsf_emb`)** — brand-context fit and suitability. v2 makes this explicit so brand-safety vendors can publish dedicated models with first-class status.
- **Inventory / Supply Embedding (`inv_emb`)** — supply-chain attributes, sellers.json signal, supply-path priors, ad server provenance.
- **Outcome Embedding (`out_emb`)** — durable, post-conversion learned representation downstream of the ranker, feeding back into `rnf_emb` updates over time.
- **World-State Embedding (`wst_emb`)** — predictive latent state from a world-state predictor (e.g., JEPA-class) sitting between the encoders and the ranker. Architectural slot — see [docs/01-architecture.md](docs/01-architecture.md) for the four-phase framework.
- **Event Sequence Embedding (`evt_seq_emb`)** — variable-length ordered sequences of event embeddings per the sequence-learning state of the art (HSTU-class). Wire format extension required to carry jagged tensors.

v2 also expands several v1 spaces' specifications:

- `phy_emb` adds richer multimodal inputs (commute patterns, neighborhood character, point-in-time mobility) beyond the v1 cell + device baseline.
- `tmp_emb` adds session-dwell and within-session recency dimensions beyond cyclical time encoding.
- `rwc_emb` adds explicit event taxonomies and a published lookup of event-class to embedding behavior.

### 8.1 Regime taxonomy (companion to v1, governed separately)

Every embedding in v1 carries a `regime_label` from a closed, versioned taxonomy (§6.6). The taxonomy itself is a companion artifact that ORP governance maintains alongside the wire-format spec.

The regime taxonomy:

- Is **closed** — labels are enumerated, not free text.
- Is **versioned** — each released version has a stable URI and is referenced explicitly by `regime_taxonomy_uri` in the bid request.
- Is **hierarchical** — labels are namespaced by their dimension (`context.university-move-in-spike`, `physical.home-relaxed`, `temporal.weekday-afternoon`, `reinforcement.selective-engaged`). The dimension prefix matches the embedding type it qualifies.
- Is **extensible** — new labels are added through the RFC process; once added, they are stable (labels are not removed within a major version).
- Carries **definitional metadata** per label — a plain-language description, illustrative examples, and any inclusion criteria.

v1.0 of the regime taxonomy ships with an initial label set per dimension. Subsequent minor versions add labels; major versions may restructure. Operators using a specific taxonomy version are immune to changes in later versions until they explicitly upgrade.

Governance of the regime taxonomy follows §18 — the same RFC-based process as the wire-format spec.

## 9. Wire format

### 9.1 OpenRTB extension

ORP rides on OpenRTB 2.6+ as a vendor extension under `ext.orp`. Example BidRequest excerpt — note that every embedding ships with a `runtime_caption`, a `regime_label`, and (where appropriate) a cell identifier; the context suite contributes multiple distinct embeddings from different sources:

```json
{
  "id": "...",
  "imp": [ { "id": "1", "..." : "..." } ],
  "user": {
    "id": "...",
    "ext": {
      "orp": {
        "v": "0.1",
        "regime_taxonomy_uri": "https://orp.example/taxonomies/regime-v1",
        "embeddings": [
          {
            "type": "uid_emb",
            "source_id": "liveramp.ats",
            "model_agreement_key": "lr-ats-uid-v3-1024",
            "dim": 1024,
            "dtype": "float16",
            "norm": "l2",
            "vector": "<base64 encoded float16 array>",
            "runtime_caption": "Returning user with established travel and lifestyle engagement; medium recency.",
            "regime_label": "user.engaged-lifestyle",
            "provenance_uri": "https://lr.example/provenance/lr-ats-uid-v3-1024",
            "caption_template_uri": "https://lr.example/templates/lr-ats-uid-v3-caption"
          },
          {
            "type": "med_emb",
            "source_id": "etherdata.moment",
            "model_agreement_key": "ether-moment-v1-128",
            "dim": 128,
            "dtype": "float16",
            "norm": "l2",
            "vector": "<base64...>",
            "runtime_caption": "Wednesday afternoon, Morningside Heights, 22°C clear. Columbia in move-in week. PS 36 in session, dismissal at 14:50. Block-party permit on 115th St. Restaurants, grocers, citibike active.",
            "regime_label": "context.university-move-in-spike",
            "provenance_uri": "https://etherdata.example/provenance/ether-moment-v1-128",
            "caption_template_uri": "https://etherdata.example/templates/moment-caption-v1",
            "composite_inputs": ["place_h3r8", "calendar_layers", "weather_observation", "school_session", "transit_state", "permits", "poi_activity"]
          },
          {
            "type": "phy_emb",
            "source_id": "publisher.device",
            "model_agreement_key": "orp-ref-phy-v1-256",
            "dim": 256,
            "vector": "<base64...>",
            "runtime_caption": "Phone, indoors, stationary; residential cell; evening lighting.",
            "regime_label": "physical.home-relaxed",
            "spatial_binding": {
              "cell_system": "h3",
              "cell_resolution": 8,
              "cell_id": "882a107293fffff"
            },
            "provenance_uri": "https://pub.example/provenance/orp-ref-phy-v1-256"
          },
          {
            "type": "tmp_emb",
            "source_id": "orp.reference",
            "model_agreement_key": "orp-ref-tmp-v1-128",
            "dim": 128,
            "vector": "<base64...>",
            "runtime_caption": "Wednesday 15:18 local; afternoon dwell window; mid-week, mid-semester academic cycle.",
            "regime_label": "temporal.weekday-afternoon",
            "provenance_uri": "https://orp.example/provenance/orp-ref-tmp-v1-128"
          },
          {
            "type": "soc_emb",
            "source_id": "liveramp.household",
            "model_agreement_key": "lr-household-v2-512",
            "dim": 512,
            "consent_id": "tcf2-...",
            "vector": "<base64...>",
            "runtime_caption": "Adult-only household indication; no co-presence proximity detected.",
            "regime_label": "social.solo-adult",
            "provenance_uri": "https://lr.example/provenance/lr-household-v2-512"
          },
          {
            "type": "rwc_emb",
            "source_id": "sportsdata.live",
            "model_agreement_key": "sd-realworld-v1-256",
            "dim": 256,
            "vector": "<base64...>",
            "runtime_caption": "No salient real-world event at this place / time. Weather stable.",
            "regime_label": "realworld.unremarkable",
            "provenance_uri": "https://sd.example/provenance/sd-realworld-v1-256"
          },
          {
            "type": "rnf_emb",
            "source_id": "publisher.firstparty",
            "model_agreement_key": "pub-rnf-v1-256",
            "dim": 256,
            "consent_id": "tcf2-...",
            "vector": "<base64...>",
            "runtime_caption": "Mixed recent advertising response; positive signal on Mediterranean food category; negative on auto.",
            "regime_label": "reinforcement.selective-engaged",
            "provenance_uri": "https://pub.example/provenance/pub-rnf-v1-256"
          }
        ]
      }
    }
  }
}
```

Notes on this example:

- **`med_emb` in this example is a composite embedding** — Ether Data's E_moment-style place × time × world-state encoding that subsumes parts of what would otherwise live in `phy_emb`, `tmp_emb`, and `rwc_emb`. Its `composite_inputs` declaration lists which canonical input pipelines it consumed. The same bid request still carries elemental `phy_emb`, `tmp_emb`, `rwc_emb`, and `soc_emb` from other providers; the ranker chooses how to combine.
- **Every embedding has a `runtime_caption`.** These are deterministic templated renderings, not LLM generations. They are the interpretability surface.
- **Every embedding has a `regime_label`** from the regime taxonomy referenced at the top of the `ext.orp` block.
- **The `phy_emb` carries a spatial binding** with H3 R8 cell ID. Other embeddings could reference the same cell ID if they're spatially scoped.

Not every bid request carries all seven user-side embeddings; emit what you have, with appropriate provenance.

### 9.2 Buyer-side embeddings (`cre_emb`, `aud_emb`)

`cre_emb` and `aud_emb` typically do not travel inside the bid request. They are loaded by the bidder when ranking candidates. ORP defines their schema (§7.3, §7.4) so that buyers and their vendor partners can exchange them out-of-band consistently.

For platforms where the bidder needs to advertise an `aud_emb` to the SSP (e.g., for SSP-side traffic shaping or pre-filtering), the same schema applies, transported in `bid.ext.orp.aud_emb` of the BidResponse.

### 9.3 Compactness and bandwidth

A 1,024-dim float16 vector is ~2KB raw, ~2.7KB base64. A bid request carrying three embeddings adds ~6–8KB to the payload. This is non-trivial. Implementers SHOULD:

- Default to float16, not float32.
- Default to 256 or 512 dim where ranker performance allows.
- Use HTTP/2 or gRPC transport with compression where possible.
- Consider Top-K sparse encoding for high-dim embeddings where appropriate.

### 9.4 Streaming and incremental updates

For applications where bid-time inclusion of full embeddings is too expensive, ORP defines an out-of-band exchange where embeddings live in a shared embedding store and the bid request carries only **embedding pointers** (`source_id` + `entity_id` + `model_agreement_key`). The bidder resolves the pointers against its cache. This pattern is RECOMMENDED for `uid_emb` where the user's vector is durable across requests.

## 10. Identity binding

ORP does not introduce new user identifiers. Each embedding is keyed to an existing identifier under `entity_binding`:

```json
"entity_binding": {
  "identifier_type": "rampid",
  "identifier_value": "<RampID>",
  "scope": "first-party-publisher" 
}
```

Supported identifier types in v1: `rampid`, `uid2`, `euid`, `openpass`, `publisher_first_party`, `retailer_first_party`, `hashed_email_sha256`, `hashed_email_md5`, `did_session` (device-scoped session, no cross-device).

**Required vs. optional bindings per embedding type:**

| Embedding | Binding scope |
| --- | --- |
| `uid_emb` | REQUIRED, user-scoped |
| `rnf_emb` | REQUIRED, user-scoped, with consent assertion |
| `soc_emb` | REQUIRED, user- or household-scoped, with consent assertion |
| `med_emb` | NOT REQUIRED (impression- or session-scoped) |
| `phy_emb` | OPTIONAL session- or device-scoped binding; raw location data MUST NOT appear here |
| `tmp_emb` | NOT REQUIRED (impression-scoped) |
| `rwc_emb` | NOT REQUIRED (global / impression-scoped) |
| `cre_emb` | NOT REQUIRED (creative-scoped) |
| `aud_emb` | NOT REQUIRED (campaign-scoped) |

## 11. Privacy, consent, and audit

### 11.1 Consent model

ORP composes with existing consent frameworks: TCF v2.2 (Europe), GPP (Global Privacy Platform), US state-specific opt-out signals. Each embedding MUST declare:

- `consent_id` — pointer to the relevant TCF/GPP/US opt-out string.
- `jurisdiction` — declared at the embedding level, not just the bid-request level (an embedding may have been computed in a jurisdiction where the user has different consent than the impression jurisdiction).
- `purposes` — TCF-compatible purposes the embedding is consented for.
- `ttl_seconds` — embedding-level expiry.

### 11.2 Right to erasure

Each `source_id` MUST publish an erasure endpoint. When a user invokes their right to erasure under GDPR, CCPA, or equivalent law, the source MUST be able to invalidate all embeddings keyed to that user across all downstream consumers. This is enforced via the `embedding_lineage_id` in the provenance manifest.

### 11.3 Provenance manifest

Every embedding MUST be accompanied by a **provenance manifest** — a structured, human- and machine-readable document hosted at the `provenance_uri`. The manifest answers a compliance reviewer's questions without requiring inspection of the model.

Minimum fields:

| Field | Description |
| --- | --- |
| `manifest_version` | Spec version of the manifest itself |
| `embedding_type` | One of the v1/v2 types |
| `embedding_class` | `"elemental"` or `"composite"` (§6.7) |
| `composite_inputs` | If `composite`, an enumerated list of the canonical input pipelines the embedding consumed |
| `model_agreement_key` | Match against the embedding's key (§12.2) |
| `model_lineage` | What model architecture, what training data, what training objective |
| `data_provenance` | What underlying data populations the embedding is derived from |
| `inference_class` | `"measured"`, `"inferred"`, or `"probabilistic"` — declares whether the underlying signal is observed directly or inferred from proxies. Captions referencing inferred / probabilistic dimensions MUST hedge appropriately (e.g., "estimated high concentration of...") |
| `caption_template_uri` | URL of the deterministic caption template the encoder co-publishes |
| `regime_taxonomy_uri` | URL of the closed, versioned regime taxonomy this embedding draws labels from |
| `activation_rate` | Fraction of impressions in a representative population where this embedding is non-null / non-default. Sparsely-activated features (`<0.1`) MUST be evaluated under conditional signal density (§12.5) |
| `null_semantics` | How the ranker should interpret a missing instance — `"absent_as_signal"`, `"absent_as_null"`, or `"absent_as_baseline"` |
| `consent_purposes` | TCF-compatible purposes |
| `jurisdictional_scope` | Where the data was collected, where the model was trained |
| `human_readable_description` | Plain-language summary suitable for legal/compliance review |
| `population_excluded` | Populations explicitly excluded from training (children, sensitive categories) |
| `update_cadence` | How often the embedding is recomputed |
| `attestation` | Cryptographic signature from the source operator |
| `conformance_attestations` | Per-gate conformance attestations referencing the LSS gates (§12.5) the embedding has passed |

**This solves the "two arrays of numbers and a score" audit problem.** Compliance reviewers, brand legal teams, and regulators can audit what an embedding represents without reading vectors. Engineers consume the same manifest programmatically to enforce contractual restrictions on use.

### 11.4 Differential privacy and aggregation

For `rnf_emb` and `out_emb` in particular, ORP RECOMMENDS that source operators apply differential-privacy noise to embeddings derived from small populations to defend against membership-inference attacks. Implementations MAY declare `dp_epsilon` in the manifest.

## 12. Model interoperability

This is the hardest problem in the spec. Two systems can compare embeddings only if their embeddings live in the same space. ORP addresses this in three layers.

### 12.1 The problem

Comparing two embeddings produced by different models produces a number, but the number has no defensible meaning. Three failure modes:

1. **Different model architectures** — a behavioral two-tower user embedding compared to a text-LLM audience embedding are in different spaces. Similarity scores are noise.
2. **Different training objectives** — two two-tower models trained on different ranking objectives produce different spaces, even at the same dimensionality.
3. **Different versions of the same model** — a v2-trained user embedding is not directly comparable to a v3 audience-query embedding from the same vendor.

### 12.2 Model agreement keys

Every embedding carries a `model_agreement_key` — an opaque identifier that uniquely names the (model, version, training-objective, training-data-snapshot) tuple. Two embeddings are *directly comparable* if and only if their `model_agreement_key` values match.

Model agreement keys are namespaced (`vendor.model_family.version`). Examples:

- `liveramp.ats-user-behavioral.v3-1024`
- `gracenote.ctv-context.v2-512`
- `bedrock.audience-query.v1-1024`

### 12.3 Cross-space ranking

ORP recognizes that requiring all systems to share a single model is impractical. The spec defines three valid configurations for ranking across heterogeneous embedding spaces:

**Configuration A — Shared model.** Vendors agree to use a common foundation model (e.g., a community-trained foundation embedding model). All `uid_emb` and `aud_emb` instances share a `model_agreement_key`. Simplest, most restrictive. Most likely path for early adoption.

**Configuration B — Translation layer.** A ranker MAY include a learned cross-encoder that maps embeddings from one space to another. The translation is itself a learned model with its own `model_agreement_key`. Translation quality MUST be declared via standard metrics (recall@K against a held-out ground-truth set) in the translation manifest.

**Configuration C — Direct cross-encoder ranking.** A ranking model MAY take embeddings from different spaces as separate inputs and learn to combine them end-to-end without translating. The ranker itself becomes the model-agreement target. The user-side and buyer-side embeddings are no longer required to be in the same space.

Configurations B and C are more flexible but require the ranker operator to publish a `ranker_manifest` describing the architecture and the embedding inputs it expects. This is the price of flexibility.

### 12.4 Compatibility negotiation

When a bidder receives a bid request with embeddings under unknown `model_agreement_key`s, it MAY:

- Reject the request (no inference possible).
- Fall back to non-embedding signals (segments, contextual taxonomies).
- Use a configured translation (Config B) if one is published.
- Use a ranker that natively accepts the unknown-space embedding (Config C).

The bidder's choice is logged in the BidResponse extension for auditability.

### 12.5 Conformance gates — what an LSS-conformant embedding must clear

Model agreement (§12.2) tells consumers whether two embeddings share a space. It does not tell them whether either embedding is *honest*. Latent Space Specifications (LSS) — the formal definitions of each first-class space — therefore include a gate framework. An embedding is **LSS-conformant** only when it passes every gate that applies to its space.

ORP defines five gate categories. Each LSS specifies which gates apply, what evaluation set is used, and what thresholds qualify as passing. Conformance attestations are recorded in the provenance manifest and the LSS publishes the methodology for each gate.

| Gate | What it tests | Why it matters |
| --- | --- | --- |
| **Downstream-Variable (DV) ablation** | Re-train the embedding with all DV-history features explicitly dropped. Lift on a held-out task must not collapse. | Prevents the embedding from secretly memorizing the outcome it claims to predict; closes the most common leakage path in ad-tech embeddings. |
| **Conditional signal density** | For every input feature contributing to the embedding, compute ΔR² *only on the subset of impressions where that feature is activated*. Sparse features (`activation_rate < 0.1`) must show conditional ΔR² ≥ 5× unconditional ΔR². | A sparsely-activated feature looks like noise when evaluated unconditionally; the gate forces honest evaluation on the population where the feature actually fires. |
| **Cross-context generalization** | The embedding must produce comparable lift in at least two structurally independent contexts (e.g., two metros, two channel types, two seasonal regimes). | Single-context lift is overfit by construction; cross-context lift is the floor for a canonical signal. |
| **Spatial equity** | Lift must not be concentrated in dense / well-resourced cells. Per-cell-class ΔR² distributions must satisfy a published equity threshold (e.g., the ratio between the top and bottom quintiles must not exceed a stated bound). | Without this gate, an embedding can add measured lift while *compounding* existing measurement inequity — adding signal where signal was already abundant and adding none where it was scarce. |
| **Architecture-specific ablation** | For learned-encoder embeddings, specific ablations the LSS requires (e.g., for JEPA-class encoders: train with the cell token masked to a constant; the masked version must still beat the simplest static baseline). | Catches architecture-specific failure modes — most importantly identity memorization disguised as moment understanding. |

Gates 1–3 are universal. Gates 4 and 5 apply when the LSS specifies them. A new gate category may be added to v2 of this section via the RFC process if a class of failure modes emerges that the existing five do not catch.

#### What conformance attestation looks like in the provenance manifest

Each embedding's provenance manifest includes a `conformance_attestations` block:

```json
"conformance_attestations": {
  "lss_version": "med_emb-v1.0",
  "gates_evaluated": ["dv_ablation", "conditional_signal_density", "cross_context", "spatial_equity"],
  "gates_passed": ["dv_ablation", "conditional_signal_density", "cross_context", "spatial_equity"],
  "evaluation_dataset_uri": "https://orp.example/lss/med_emb-v1.0/eval-2026-q2",
  "results_uri": "https://provider.example/conformance/med_emb-v1.0/2026-q2-results.json",
  "attested_by": "provider.example",
  "attestation_signature": "..."
}
```

Conformance attestations are auditable: receivers can verify a provider's claim by re-running the evaluation against the published dataset. Conformance is *not* a one-time certification — LSSs may require re-attestation on a published cadence (typically per major model version or per quarter, whichever is more frequent).

## 13. Versioning

ORP follows semantic versioning: MAJOR.MINOR.PATCH.

- **MAJOR** — incompatible wire-format changes.
- **MINOR** — new embedding types, new optional fields. Backwards compatible.
- **PATCH** — clarifications, typo fixes, no semantic change.

`model_agreement_key`s are independent of ORP versioning and managed by each vendor.

## 14. Reference implementation

A reference implementation is planned, structured as:

- `/spec` — this document and its successors
- `/schemas` — JSON Schema files for each embedding type and the provenance manifest
- `/transport` — OpenRTB extension reference, gRPC IDL, sample wire payloads
- `/sdks` — TypeScript and Python client libraries for emitting and consuming embeddings, validating provenance manifests, negotiating model agreement
- `/prebid-module` — Prebid integration for browser-edge `uid_emb` and `ctx_emb` emission, modeled on the ATS.js + Prebid pattern from Agentic Audiences
- `/sidecar` — container image for in-loop ranking inside ARTF environments, with model-agreement negotiation built in

The reference implementation will be developed in parallel with the spec under the same licenses.

## 15. Security considerations

### 15.1 Embedding inversion

A sufficiently expressive embedding can be inverted to reconstruct properties of the underlying user. Source operators SHOULD:

- Apply differential-privacy noise where the embedding is derived from small populations (§11.4).
- Avoid embedding sensitive categories directly. Sensitive categories should be filtered upstream of the embedding model, not encoded into it.

### 15.2 Membership inference

A bidder receiving an embedding MAY attempt membership-inference attacks (does this embedding belong to a specific known individual?). Mitigations:

- TTL-bounded embeddings.
- DP-noised reinforcement embeddings.
- Embedding lineage IDs that allow erasure propagation.

### 15.3 Replay attacks

`ctx_emb` and `rnf_emb` MUST carry signed timestamps in the provenance manifest. Bidders SHOULD reject embeddings whose timestamps fall outside a tight window.

### 15.4 Provenance forgery

Provenance manifests MUST be served from the source operator's domain over HTTPS and carry a cryptographic attestation. Implementations SHOULD pin source-operator keys via the Agent Registry pattern from IAB Tech Lab's [AAMP](https://iabtechlab.com/standards/aamp-agentic-advertising-management-protocols/).

## 16. Relationship to existing specs

ORP composes with, rather than replaces, the existing open-web stack.

| Existing spec | Relationship |
| --- | --- |
| **OpenRTB 2.6+** | Transport. ORP rides as a vendor extension. |
| **IAB Tech Lab AAMP** | ORP is intended to fit under AAMP's "Agentic Protocols" pillar. |
| **ARTF** | Execution substrate. ORP's reference sidecar runs inside ARTF environments. |
| **Agentic Audiences (UCP)** | Predecessor and primary reference. ORP extends UCP's three signal types to five first-class embeddings, formalizes model interoperability, and adds the provenance manifest. ORP and Agentic Audiences SHOULD become a single specification if and when the IAB Tech Lab community agrees. |
| **IAB Audience Taxonomy v1.1** | Coexists. The bid request carries both taxonomy segment IDs and ORP embeddings. |
| **Agent Registry** | Used for source-operator attestation (§15.4). |
| **Buyer Agent / Seller Agent SDKs** | ORP `aud_emb` is what these agents exchange when they go beyond text prompts. |
| **Prebid** | Reference browser-edge implementation borrows the ATS.js + Prebid pattern. |
| **OpenDirect, Deals API, AdCOM** | Out of scope. ORP is RTB-side. |
| **GPP / TCF v2.2** | Composes via the `consent_id` field. |

## 17. Open questions

- **Naming.** ORP is a placeholder. Community should converge on a final name.
- **Granularity of the context suite.** v1 defines five context-facet embeddings (`med_emb`, `phy_emb`, `tmp_emb`, `soc_emb`, `rwc_emb`). This is a design choice. Some implementations may want to merge facets (e.g., physical + temporal into a single "situation" embedding); others may want to split further (e.g., separate "place semantics" from "device" inside `phy_emb`). The current split is meant to mirror what production walled-garden rankers actually expose as inputs. Community input is invited.
- **Reference temporal embedder.** v1 mentions that an open, deterministic temporal embedder should be published as a reference so every operator can produce comparable `tmp_emb` instances. Who builds it is open.
- **`soc_emb` consent and jurisdiction.** Social-context signal touches household graphs and co-presence, which raise privacy issues that vary sharply by jurisdiction. Whether `soc_emb` belongs in v1 at all, or should be deferred to v2 with a stronger consent model, is open.
- **Foundation model availability.** Configuration A in §12 depends on a credible neutral foundation model being available. Whether this comes from open-source (e.g., a community-trained model under permissive license), a vendor donation (LiveRamp post-Publicis, or another vendor), or a hyperscaler, is open.
- **Reinforcement-embedding governance.** `rnf_emb` is the most privacy-sensitive embedding. Whether its inclusion in v1 is correct, or whether it should be deferred to v2 with a stronger consent model, is open.
- **Browser-edge vs. server-side emission.** ATS.js + Prebid uses browser-edge emission. Server-side emission via supply-side platforms is also valid. The trade-offs are non-trivial.
- **Embedding stores and pointer-based bidding.** §9.4 mentions out-of-band embedding stores. The exact protocol for cache invalidation, freshness guarantees, and access control is open.
- **Audit attestations.** Whether provenance manifests should be additionally attested by a third-party auditor (think TAG, NAI, BPA) is open.
- **Coordination with the Publicis-LiveRamp transaction.** As of mid-2026, LiveRamp's neutrality posture is being re-evaluated post-acquisition. ORP's relationship with Agentic Audiences may need to evolve depending on how that plays out.

## 18. Governance

This v0.1 draft is published independently to invite community discussion. The intended long-term home is a neutral standards body. Options under consideration:

- **IAB Tech Lab AAMP working group.** Most natural fit given the relationship to Agentic Audiences, ARTF, and the buyer/seller agent SDKs.
- **W3C.** Possible if the protocol expands toward browser-API integration.
- **Independent foundation.** Possible if a coalition of operators prefers operator-led governance.

The author's preference is IAB Tech Lab AAMP contribution, in the spirit of how LiveRamp donated UCP.

Until the spec finds a permanent home, governance is:

- Pull requests welcome under the GitHub repo where this spec is published.
- A draft Commit Group will be convened from interested operators (DSPs, SSPs, identity providers, data providers, publishers, agencies).
- Major design decisions will be made via RFC-style proposals with public comment periods of at least 14 days.

## 19. Glossary

- **Activation rate** — fraction of impressions in a representative population where an embedding is non-null. Sparsely-activated embeddings (`<0.1`) must be evaluated under conditional signal density.
- **Audience query** — the buyer's expression, in natural language or seed-audience form, of who they want to reach in this moment.
- **Bid request** — the OpenRTB request from the SSP/exchange to the DSP.
- **Caption template** — deterministic, slot-filling template that renders a structured input vector into a plain-language sentence. Published alongside the embedding encoder and consumed by the runtime to produce per-instance captions.
- **Composite embedding** — an embedding produced by a structured product of multiple input pipelines, often spanning multiple ORP dimensions (see §6.7). The provenance manifest's `composite_inputs` field enumerates which canonical input pipelines were consumed.
- **Conformance gate** — one of the testable properties an embedding must clear to be LSS-conformant (§12.5). Five categories: DV ablation, conditional signal density, cross-context generalization, spatial equity, architecture-specific ablation.
- **Context suite** — the five first-class embeddings in v1 that describe distinct facets of the moment (`med_emb`, `phy_emb`, `tmp_emb`, `soc_emb`, `rwc_emb`). ORP treats context as a suite, not a single representation.
- **Cross-encoder** — a model that takes two or more inputs and learns to combine them directly without projecting them to a shared space first.
- **Embedding** — a learned dense vector representation of an entity.
- **Embedding store** — a cache/database holding pre-computed embeddings keyed by entity identifier.
- **Identity binding** — the link between an embedding and a user identifier (RampID, UID2, etc.).
- **Media context** — what is being consumed in the moment: page, show, app screen, podcast episode, surrounding creative.
- **Model agreement key** — opaque identifier naming the (model, version, training-objective, training-snapshot) tuple. See §12.2.
- **Multi-input architecture** — a ranking-model architecture in which separate inputs (user, each context facet, creative, audience query, reinforcement) feed into a combining model. The walled-garden production pattern.
- **Multi-tower architecture** — a special case of multi-input in which each input has its own tower producing an embedding, with a final layer combining them.
- **Physical context** — where and how the impression occurs: device, form factor, motion, environment, place semantics.
- **DV (downstream variable)** — an outcome or near-outcome signal (clicks, conversions, panel response). DV-history features are excluded from the embedding to prevent label leakage; the DV-ablation gate enforces this discipline.
- **Elemental embedding** — an embedding produced by a single encoder over a coherent input pipeline (see §6.7). Contrast with composite embedding.
- **H3** — Uber's hexagonal hierarchical geospatial grid system. ORP's RECOMMENDED default spatial identifier system for place-scoped state, at resolution 8 (~460 m hexagonal edge).
- **L1 / L2 / L3 layers** — the canonical / consumer / operational layering used by some collaborator stacks (Ether Data; see [docs/01-architecture.md](docs/01-architecture.md)). L1 = universal canonical form and national lookups; L2 = consumer-specific ranker (DV-aware); L3 = local operational deployment.
- **Latent Space Specification (LSS)** — the formal definition of a single first-class embedding space: semantic scope, axes of variation, invariants, conformance gates, evaluation benchmarks. Published companion to the wire-format spec; voluntarily adopted; required for cross-vendor interop without explicit translation layers.
- **Provenance manifest** — the human- and machine-readable document accompanying each embedding (§11.3).
- **Ranker** — the learned function that consumes embeddings and produces a score.
- **Real-world context** — what is happening in the world around the impression: events, weather, news, market moves.
- **Regime label** — a categorical compression of the embedded moment drawn from a closed, versioned taxonomy. Carried per embedding instance in the bid request; queryable independently of the vector.
- **Reinforcement signal** — a representation of how a user has responded to recent advertising.
- **Runtime caption** — the per-impression plain-language rendering of an embedding's inputs, produced by the caption template upstream of (and from the same inputs as) the encoder. Required for every first-class embedding instance.
- **Social context** — who is around the user and what social setting the impression sits in: co-presence proximity, household / cohort signal.
- **Temporal context** — when the impression occurs, encoded for direct use by the ranker: time-of-day, day-of-week, recency, seasonality, dwell.
- **Two-tower architecture** — a special case of multi-tower with two towers, typically user and item.

---

## Appendix A — Worked example: the Italy ad

A concrete illustration of how ORP works in practice, and why context has to be a *suite* rather than a single embedding.

**Scene.** A user spends three hours at a friend's dance-studio gala. The friend, sitting at the same table, has been searching for retirement villas in Italy. Later that evening, the user opens Instagram on the couch and sees an ad for a Tuscan villa.

**What rode the bid request:**

User-side:
- `uid_emb` — the user's durable behavioral embedding from their identity provider. Encodes that they engage with travel, design, and family content. Has nothing specifically about Italy.
- `rnf_emb` — the user's reinforcement signal: they have ignored most travel ads recently but engaged with a couple of Mediterranean food ads.

Context suite — five distinct facets of the moment, *each its own embedding*:
- `med_emb` — Instagram feed, lifestyle-content adjacency, currently surrounded by other people's vacation photos.
- `phy_emb` — phone, indoors, stationary, low-light environment consistent with evening at home.
- `tmp_emb` — late evening on a weekend, end of a long session, dwell pattern consistent with relaxed scrolling.
- `soc_emb` — co-presence proximity earlier in the evening to a household graph node who has been searching for Italian retirement villas. This is the embedding doing the heaviest lift, and it is one of the embeddings the open web has never had.
- `rwc_emb` — no salient real-world event; effectively a no-op in this moment, which is itself signal.

Buyer-side (not in the bid request, loaded by the bidder when scoring candidates):
- `aud_emb` — the campaign's query: "U.S. adults with disposable income, currently in-market or proximate-to-in-market for Italian travel."
- `cre_emb` — the particular Tuscan villa rental ad.

**How the ranker uses them.** None of the embeddings by itself is decisive. The user's identity embedding is moderately compatible with the audience query. The reinforcement signal is mixed but the Mediterranean food signal is positive. Media context (lifestyle feed, vacation photos surrounding) is favorable to travel creative. Physical and temporal context (phone, evening, stationary, relaxed dwell) suggest a high-quality engagement window. Real-world context contributes nothing here, which is fine.

The signal that puts this impression over the bidding threshold is `soc_emb`. The user's co-presence proximity earlier in the evening to someone actively searching for Italian retirement villas, combined with the audience query expression around Italian travel, gives the ranker a high-confidence inference that this is a meaningful moment for this campaign.

The bid wins. The ad serves.

No one was listening. The system understood.

This is the multi-dimensional ranker the walled gardens have run for a decade. The open-web's parity move depends on every link in that chain — identity, *each facet of context separately*, creative, audience query, reinforcement — being able to travel across vendor boundaries with defensible meaning. That is what ORP defines.

**The point of the context suite.** If `med_emb`, `phy_emb`, `tmp_emb`, `soc_emb`, and `rwc_emb` had been collapsed into a single "context embedding," the ranker would lose the ability to weight social proximity differently from media adjacency from temporal dwell from real-world salience. The whole bet of multi-dimensional ranking is that *each dimension carries different signal to different campaigns at different moments*. Collapsing them flattens the bet.
