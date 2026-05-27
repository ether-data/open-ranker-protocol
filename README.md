# Open Ranker Protocol (ORP)

**An open standard for shared latent spaces and multi-provider embedding exchange in real-time ad decisioning.**

> Identity graphs defined the last era of advertising. Latent world-state models and contextual intelligence will define the next one. ORP is the open substrate that lets the open web run that next era — without ceding the semantic ground to any single operator.

---

## What ORP is

ORP is three things at once:

1. **A wire format** — an OpenRTB extension that carries embeddings and structured signal envelopes through the bid request alongside existing taxonomies and IDs.
2. **A shared latent space ontology** — formal definitions of *what each embedding dimension represents*, so multiple providers can train independent models that produce interoperable embeddings.
3. **A multi-provider runtime model** — a contract that lets identity providers, context providers, creative providers, and audience providers each insert their embeddings into a single ad decision at bid time, with results interpretable by both advertisers and publishers.

The architectural ambition is captured in a single line: ORP makes the open web capable of bid-time reasoning across multiple latent spaces simultaneously, the way walled-garden rankers already do internally.

## The problem ORP solves

The open web has three categories of audience representation today: identity (RampID, UID2), taxonomies (IAB Audience Taxonomy), and contextual signals (page metadata). All three are static, single-source, and combined outside the ranker via Boolean targeting.

Walled-garden rankers don't work like this. They run learned models that combine many distinct embedding inputs simultaneously — user, content, creative, context, surrounding state — and they recalculate the combination on every impression. That architecture is publicly documented ([Google Ad Rank inputs](https://support.google.com/google-ads/answer/1722122), [Meta DLRM](https://github.com/facebookresearch/dlrm), [Meta sequence learning](https://engineering.fb.com/2024/11/19/data-infrastructure/sequence-learning-personalized-ads-recommendations/)), running on custom silicon ([Meta MTIA 2i](https://ai.meta.com/blog/next-generation-meta-training-inference-accelerator-AI-MTIA/)), and ~44% cheaper per inference than the open web's equivalent on commercial GPUs.

The performance gap between the open web and the walled gardens is not primarily a data gap. It's a **substrate gap** with three layers:

| Layer | Walled-garden state | Open-web answer |
| --- | --- | --- |
| **Inputs** | Multi-dimensional, including media, physical, temporal, social, real-world context | Static identity + taxonomy + page context |
| **Architecture** | Learned ranker combining many embeddings at decision time | Boolean targeting against segments |
| **Latent semantics** | Single owner defines all spaces internally | No shared semantic ground across vendors |

ORP closes the last row first. Defining shared latent spaces is the foundational move; the wire format and the runtime model follow from it.

## The four-phase architecture

ORP is designed against a four-phase architectural target, even though only the first two phases need to ship in v1 to make the standard useful.

| Phase | What it builds | Status |
| --- | --- | --- |
| **1. Context Foundation** | Multimodal context envelope: H3 spatial structure, temporal dynamics, POI composition, census/socioeconomic layers, events, mobility, weather, content metadata, device/inventory metadata. Continuously refreshed latent representation of the environment around an impression. | v1 |
| **2. Cross-Modal Alignment** | Separate encoders for context, creative, campaign intent, response history — projected into *compatible* latent spaces that a ranker can interact with jointly. | v1 |
| **3. World-State Modeling** | Latent state prediction layer (e.g., JEPA-style) that infers missing signals, predicts future contextual states, and models transitions. Sits *before* the ranker, not as the ranker. | v2 |
| **4. Ranking + Reinforcement** | Bid-time inference layer combining all inputs: `score = f(identity, context, creative, campaign, reinforcement)`. Evolves from gradient boosting to adaptive online ranking. | Operator-implemented, ORP-compatible |

See [docs/01-architecture.md](docs/01-architecture.md) for full architecture detail.

## What's open, what's proprietary

ORP draws a deliberate line: **the latent-space ontology, the envelope format, and the runtime contract are open**. The continuously-trained world-state models, the proprietary embeddings that conform to the spec, and the ranking systems built on top are commercial.

This is the same pattern OpenRTB used to draw between protocol and platform.

| Open (ORP standard) | Proprietary (operator/provider) |
| --- | --- |
| Wire format (OpenRTB extension) | Specific embedding models |
| Latent-space ontology per dimension | Training data and procedures |
| Context envelope structure | World-state predictors |
| Provenance manifest schema | Ranking models |
| Conformance benchmarks | Inference infrastructure |
| Identity binding rules | Custom silicon |

See [docs/06-governance.md](docs/06-governance.md) for the full open/proprietary boundary.

## The shared latent space — the central idea

The most novel contribution of this spec is the **shared latent-space ontology**. For each embedding dimension (user identity, media context, physical context, temporal context, social context, real-world context, creative, audience query, reinforcement), ORP defines:

1. **Semantic scope** — what the space represents.
2. **Axes of variation** — what dimensions of meaning the space encodes.
3. **Invariants** — testable properties any conformant embedding must satisfy.
4. **Reference evaluation benchmarks** — standard test sets and metrics for conformance.
5. **Provenance manifest schema** — per-provider declaration of how their embedding conforms.

This is the layer that lets multiple providers contribute embeddings to the same decision call. Without it, embeddings in different spaces are incomparable, and the open web collapses to one operator's model. With it, the open web can scale to many providers each contributing to a shared ranker call.

See [docs/02-latent-space.md](docs/02-latent-space.md) for the full ontology approach.

## How a provider plugs in

A provider that wants to contribute embeddings to ORP-compliant ad decisions does four things:

1. **Trains a model** that produces embeddings conformant to one or more ORP latent-space specifications.
2. **Publishes a provenance manifest** declaring the embedding type, model lineage, training data, consent posture, and conformance attestation.
3. **Emits embeddings into the bid request** (or into a referenced embedding store) according to the wire format.
4. **Passes conformance benchmarks** — automated tests that validate the embedding's behavior on standard test sets.

That's the contract. The provider keeps their model proprietary. The ORP-compliant ranker on the buyer side knows what to do with the embedding because the latent space is defined.

See [docs/04-provider-integration.md](docs/04-provider-integration.md) for end-to-end provider integration.

## How advertisers and publishers interpret the system

Embeddings as a primitive have an audit-trail problem: vectors and similarity scores don't tell a compliance reviewer what's happening. ORP solves this with the **provenance manifest** — a structured, human- and machine-readable document accompanying every embedding that answers:

- What does this embedding represent? (semantic scope)
- What populations is it derived from? (data provenance)
- What consents does it carry? (privacy posture)
- What is its model lineage? (training history)
- What latent-space specification does it conform to? (interpretability)
- What was the decision context? (winning bid disclosure)

Both advertisers (assessing whether an ORP-driven ad served correctly) and publishers (understanding what signals their inventory is being matched against) can read the same manifest. Brand legal, agency compliance, and privacy auditors get a defensible artifact without needing to read vectors.

See [docs/05-interpretability.md](docs/05-interpretability.md) for the interpretability model.

## Repository layout

```
open-ranker-protocol/
├── README.md                    This file — landing page and orientation
├── SPEC.md                      Formal v0.1 specification (wire format, schemas, transport)
├── CONTRIBUTING.md              How to contribute to the spec and reference implementations
├── docs/
│   ├── 01-architecture.md       The four-phase architecture (Foundation → Alignment → World-State → Ranking)
│   ├── 02-latent-space.md       The shared latent-space ontology
│   ├── 03-context-envelope.md   The multimodal context envelope specification
│   ├── 04-provider-integration.md   How providers plug their embeddings into ORP decisions
│   ├── 05-interpretability.md   How advertisers and publishers read the system
│   ├── 06-governance.md         Open standard vs. proprietary models
│   └── 07-runtime-model.md      What happens at bid time, end to end
├── schemas/                     (planned) JSON Schema definitions
├── transport/                   (planned) OpenRTB extension reference, gRPC IDL, sample payloads
├── sdks/                        (planned) TypeScript and Python client libraries
└── conformance/                 (planned) Test suites for each latent-space specification
```

## The nine first-class embeddings (v1)

ORP defines nine first-class embedding spaces, organized in three groupings.

**User-side — who is here and how they have responded:**

| Embedding | What it represents |
| --- | --- |
| **User Identity** (`uid_emb`) | Who this person is, learned from durable behavioral history |
| **Reinforcement** (`rnf_emb`) | How this user has responded to recent advertising |

**Context suite — five distinct facets of the moment:**

| Embedding | What it represents |
| --- | --- |
| **Media Context** (`med_emb`) | What is being consumed — page, show, app screen, surrounding creative |
| **Physical Context** (`phy_emb`) | Where and how — device, motion, environment, place semantics |
| **Temporal Context** (`tmp_emb`) | When — time-of-day, recency, seasonality, dwell |
| **Social Context** (`soc_emb`) | Who is around — co-presence proximity, household / cohort signal |
| **Real-World Context** (`rwc_emb`) | What is happening in the world right now — events, weather, news |

**Buyer-side — what the campaign wants and what is on offer:**

| Embedding | What it represents |
| --- | --- |
| **Creative** (`cre_emb`) | The ad itself — copy, image, video, format, brand attributes |
| **Audience Query** (`aud_emb`) | What the campaign wants in this moment |

A v2 roadmap adds placement, brand-safety, inventory/supply, outcome, and event-sequence embeddings.

## Status

**v0.1 draft, open for comment.**

The spec, the latent-space ontology, and the documentation are being developed in the open under permissive licenses. The intended long-term governance home is a neutral standards body — preference is IAB Tech Lab contribution as part of the AAMP framework.

| Component | Status |
| --- | --- |
| SPEC.md v0.1 (wire format) | Draft |
| Latent-space ontology framing | Draft |
| Context envelope specification | Draft |
| Provider integration model | Draft |
| Reference evaluation benchmarks | Planned |
| JSON Schemas | Planned |
| Reference SDKs (TS, Python) | Planned |
| Prebid module | Planned |
| ARTF sidecar reference | Planned |

## License

- **Specification text:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)
- **Reference implementations:** [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Acknowledgments

Authored and edited by Nathan Woodman, with substantial architectural input from collaborators including the EtherData team (Laurent Madeleni's four-phase architecture and context-envelope framing; Yuri Khrakovsky's standards-alignment perspective) and Damian Naglak (Bedrock Platform; segments-vs-embeddings practitioner framing).

This spec builds directly on the IAB Tech Lab AAMP initiative ([ARTF](https://iabtechlab.com/standards/artf/), [Agentic Audiences](https://github.com/IABTechLab/agentic-audiences), buyer/seller agent SDKs) and on LiveRamp's original UCP donation. It draws conceptual debt from the multi-tower and sequence-learning recommendation architectures pioneered at YouTube, Meta, TikTok, and Pinterest.

## Companion reading

- [IAB Tech Lab AAMP framework](https://iabtechlab.com/standards/aamp-agentic-advertising-management-protocols/)
- [ARTF — Agentic Real Time Framework](https://iabtechlab.com/standards/artf/)
- [Agentic Audiences (formerly LiveRamp UCP)](https://github.com/IABTechLab/agentic-audiences)
- [IAB Buyer Agent SDK](https://github.com/IABTechLab/buyer-agent)
- [IAB Seller Agent SDK](https://github.com/IABTechLab/seller-agent)
