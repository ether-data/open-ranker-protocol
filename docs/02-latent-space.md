# Shared Latent Space — The LSS Framework

This document defines how ORP turns a wire format into a *shared semantic ground* — the contract that lets multiple providers train independent models whose embeddings remain interoperable.

The mechanism is the **Latent Space Specification (LSS)**: a formal, voluntarily-adopted document, one per first-class embedding type, that defines what the space represents, what properties any conformant embedding must satisfy, and what gates a provider must clear before claiming conformance.

LSS is the contribution of this project that doesn't exist anywhere else in the open-web ad-tech stack. The wire format is necessary but not sufficient. Without LSS, embeddings in different spaces are incomparable and the open web collapses to one operator's model. With LSS, the open web scales to many providers contributing to one shared decisioning call.

---

## 1. Why the latent space is the central concept

A latent space is more than the geometry of the vector (dimension, dtype, normalization — the wire-format concerns). It is the *semantic structure*:

1. **What axes of variation it encodes** — what dimensions of meaning are organized along the space.
2. **What proximity means** — when two vectors are close, what is similar about the entities they represent.
3. **What invariants hold** — testable properties any conformant model must satisfy.
4. **What training-objective semantics produce the space** — what supervisory signal organizes the geometry.

For two embeddings to be *meaningfully comparable*, they need to live in the same latent space. The wire format alone cannot guarantee this. Two providers can both emit 512-dimensional L2-normalized float16 vectors and still produce embeddings whose similarity scores have no defensible meaning.

The LSS framework closes this gap.

---

## 2. The two coordination paths — model agreement vs. LSS conformance

ORP supports two complementary paths for cross-vendor embedding interoperability. Both are valid; operators can use either or both.

### Path 1 — Bilateral model agreement (SPEC §12.2)

Two parties (a provider and a consumer) agree to use the same `model_agreement_key`. Embeddings under matching keys are directly comparable *by construction* — they came from the same model.

Pros: simplest, no semantic specification required.
Cons: doesn't scale beyond bilateral relationships; locks in one model; each new model version requires renegotiation.

### Path 2 — LSS conformance (this document)

A provider trains a model that conforms to a published Latent Space Specification. The LSS defines what the space means and how to test conformance. Multiple providers can train independent models that all conform to the same LSS; their embeddings remain interoperable without any bilateral arrangement.

Pros: scales to many providers; survives model-version churn; provides a defensible audit trail.
Cons: requires the LSS to exist and be maintained; conformance testing is real work.

LSS conformance is the multilateral coordination path that lets the open web operate at ecosystem scale. Path 1 is the fallback when an LSS doesn't yet exist for a given space.

---

## 3. Anatomy of an LSS document

Each first-class embedding type in ORP has its own LSS. Every LSS document includes the same eight sections.

### 3.1 Semantic scope

A plain-language definition of what this latent space represents and — equally important — what it does not. The scope is narrow enough that conformant models cluster around the same understanding of the space.

Example for `med_emb`: *"The Media Context Embedding represents what is being consumed at the impression — the page, the show, the app screen, the surrounding creative, and the content adjacency. It does not represent user identity, physical location, time of day, social context, or real-world events; those are separate spaces with their own LSS."*

### 3.2 Axes of variation

The dimensions of meaning the space should encode. Not the literal vector dimensions (those are wire-format concerns) but the *semantic* axes a conformant model should organize the space along.

Example for `med_emb`: *"Topical content; format and modality; freshness; length / consumption duration; brand-safety tier; content tone; audience-suitability category."*

Axes are not exhaustive — they are the dimensions that conformance tests will probe.

### 3.3 Invariants

Testable properties any conformant embedding must satisfy. These are the assertions a conformance test suite verifies.

Examples for `med_emb`:
- Two pages about the same news event should be closer to each other than to an unrelated page in the same outlet.
- A long-form documentary and a short-form social video on the same topic should be moderately close but distinguishable.
- A page that is clearly off-brand-safety (e.g., explicit content) should be at least *d* away from any page in a brand-safe topic cluster.

Invariants are the operational substance of an LSS. They are what makes the space *defined* rather than vibes.

### 3.4 Training-objective guidance

The LSS does not mandate a specific training procedure — providers can innovate on training. But the LSS specifies what kind of supervisory signal the resulting space should reflect. This guidance is the bridge between "what we want the space to mean" and "what a provider has to do to get there."

Example for `med_emb`: *"The recommended supervisory signal is engagement-based content similarity (co-engagement, co-consumption) supplemented with topical-similarity contrastive loss over editorial taxonomies. Pure text-embedding similarity (sentence-transformer style) is insufficient because it does not capture format / modality / freshness axes."*

This is where the LSS pushes back on the "everything is a text embedding" failure mode. Each space's training guidance specifies what signals must shape the geometry beyond text similarity.

### 3.5 Conformance gates

The five gate categories from SPEC §12.5. Each LSS specifies which gates apply and what thresholds qualify as passing.

| Gate | Universal? | Example threshold |
| --- | --- | --- |
| **DV-free ablation** | Yes — all LSSs | Lift on held-out task must not decrease by more than 5% when DV-history features are dropped from training |
| **Conditional signal density** | Yes — all LSSs | For features with `activation_rate < 0.1`, conditional ΔR² must be ≥ 5× unconditional ΔR² |
| **Cross-context generalization** | Yes — all LSSs | Lift in two structurally independent contexts must be within a stated band (e.g., max-to-min ratio ≤ 1.5) |
| **Spatial equity** | When the space carries place-scoped state | ΔR² across cell-density quintiles must satisfy a stated maximum top-to-bottom ratio (e.g., ≤ 3.0) |
| **Architecture-specific ablation** | When applicable | Per-architecture (e.g., cell-token masked baseline must beat E_time-alone for JEPA-class encoders) |

Each LSS publishes a *reference evaluation set* against which gates are tested. Providers re-attest periodically (typically per major model version or per quarter).

### 3.6 Reference evaluation set

The LSS publishes a standard test set and the evaluation methodology for each gate. The test set is versioned and stable; the methodology is reproducible.

Example for `med_emb`: a curated set of ~10,000 web pages and ~5,000 CTV episodes with human-annotated topical clusters, engagement-similarity pairs, brand-safety labels, and freshness timestamps. The LSS publishes the dataset under a permissive license alongside the gate evaluation scripts.

Conformance evaluation is *reproducible*: any operator can re-run the gates against a provider's claimed conformant model and verify the attestation.

### 3.7 Cross-space relationships

Some LSSs declare expected relationships with other spaces. These are not strict requirements but published guarantees that downstream rankers can rely on.

Examples:
- `cre_emb × med_emb` should produce a meaningful brand-safety score when combined via a published bilinear form.
- `phy_emb × tmp_emb` should distinguish places that respond differently to the same temporal shift (the bilinear-cross construction Ether Data ships).
- `uid_emb × aud_emb` similarity should rank candidate audiences for a user with stated AUC on a held-out benchmark.

Cross-space relationships are how ORP's multi-input ranking story becomes operational. Without them, embeddings travel together but the ranker has to learn every interaction from scratch.

### 3.8 Versioning and stability

Each LSS is versioned independently. The wire format (SPEC) and each LSS evolve on independent cadences.

- **Patch versions** (`v1.0.1`) — clarifications, typos, no semantic change.
- **Minor versions** (`v1.1`) — added axes, added invariants, added cross-space relationships. Backwards-compatible.
- **Major versions** (`v2.0`) — semantic shift in what the space represents. Providers must re-attest.

The `model_agreement_key` includes the LSS version it conforms to. Operators using a specific LSS version are immune to changes in later versions until they explicitly upgrade.

---

## 4. The nine LSSs of v1

ORP v1 covers nine first-class embeddings. The LSS documents themselves are companion artifacts to this SPEC; the first batch ships alongside the v1.0 wire format. v1.0 LSS ambition by space:

| LSS | Difficulty | v1.0 readiness |
| --- | --- | --- |
| `tmp_emb` | Easy | Can ship deterministic reference encoder. Time has well-defined cyclical encodings (sin/cos transforms, recency decay). |
| `med_emb` | Medium | 15+ years of content-embedding literature. Multiple candidate models exist. |
| `cre_emb` | Medium | Similar maturity to `med_emb`. Multimodal creative-embedding work is well-established. |
| `aud_emb` | Medium | Text-LLM and seed-audience patterns both well-understood. Two-track LSS likely. |
| `phy_emb` | Hard | Place semantics × device semantics requires substantial work. H3 R8 spatial binding is settled; the learned encoder is not. |
| `uid_emb` | Hard | User identity space is contentious (behavioral vs. demographic vs. declared interest); also most privacy-sensitive. |
| `rwc_emb` | Hard | Event taxonomies exist but no learned-embedding standard. Real-time event coverage is geographically uneven. |
| `soc_emb` | Hardest | No standard latent representation in the literature. Highest privacy load. |
| `rnf_emb` | Hardest | Inherently sequence-based; the LSS work really belongs to the v2 sequence-learning track. |

v0.1 of the project ships **`tmp_emb-v1.0`** and **`med_emb-v1.0`** as the first two LSS documents (the easiest two). The remaining seven follow at a published cadence.

Operators can use ORP today without LSS conformance — `model_agreement_key` bilateral arrangements remain valid. LSS conformance is the path toward ecosystem-scale interoperability, not a precondition for adoption.

---

## 5. What an LSS document looks like in practice

A minimal LSS document is structured markdown of ~10-20 pages including:

```
# LSS — Media Context Embedding (med_emb) v1.0

## 1. Semantic scope
## 2. Axes of variation
## 3. Invariants
## 4. Training-objective guidance
## 5. Conformance gates
## 6. Reference evaluation set
## 7. Cross-space relationships
## 8. Versioning and stability
## 9. Open questions
## 10. Change log
```

Each LSS is stored in `lss/` in this repository, versioned alongside the wire format but governed independently per §3.8.

The reference evaluation set lives in `lss/<embedding_type>/eval/` and is downloadable directly. Evaluation scripts live in `lss/<embedding_type>/scripts/` and are runnable against any candidate model.

---

## 6. What an LSS is not

This is worth saying explicitly because the term "specification" can suggest more than is meant.

- **An LSS is not a mandated model.** Vendors who innovate on training, on architecture, on the data they consume, all remain valid as long as their resulting embedding passes the gates. The LSS specifies *what the space means*, not *how to build it*.
- **An LSS is not a single foundation model.** SPEC §12.3 Configuration A allows for a shared foundation model when one exists, but that is one possible Layer-3 implementation choice, not a requirement of the LSS itself.
- **An LSS is not a privacy specification.** Privacy and consent are governed by SPEC §11 (provenance manifest, consent declarations, jurisdictional scope). The LSS focuses on semantic and functional conformance.
- **An LSS is not static.** As the field evolves and as walled-garden state-of-the-art shifts (e.g., from DLRM to HSTU-class), the LSS must evolve too. The versioning discipline (§3.8) is how the LSS keeps up without forcing all operators to upgrade in lockstep.

---

## 7. Governance of the LSS layer

LSS documents are governed under the same RFC process as the wire-format spec (SPEC §18). The intent: each LSS is developed in the open by an interested working group, reviewed by the broader community, and published with stable versioning.

For an LSS to advance to canonical (v1.0+):
- The reference evaluation set must be published under a permissive license.
- At least two independent operators must have attested conformant models against the gates.
- The cross-context generalization gate must be cleared across at least two structurally independent deployment environments.

This bar is deliberately high. An LSS that doesn't clear it stays in draft and operators continue to use bilateral model agreements until it does.

---

## See also

- [SPEC.md §12](../SPEC.md#12-model-interoperability) — the formal model-interoperability and conformance-gate definitions
- [SPEC.md §6.5-6.7](../SPEC.md#65-every-embedding-instance-ships-with-a-runtime-caption) — runtime caption, regime label, composite embeddings, all of which connect to the LSS framework
- [docs/01-architecture.md](01-architecture.md) — the four phases, particularly Phase 2 (Cross-Modal Alignment) which depends on LSS to operate
- [docs/03-context-envelope.md](03-context-envelope.md) — how the LSS framework applies to the multimodal context envelope
- [docs/04-provider-integration.md](04-provider-integration.md) — how providers conform their embeddings to an LSS
- [docs/06-governance.md](06-governance.md) — open / proprietary boundary, of which LSS is the most consequential expression
