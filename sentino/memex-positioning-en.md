# Sentino Memex Memory Framework

**Product Overview & Technical Reference**

---

Document version: v1.0
Date: 2026-04-17
Intended use: customer conversations, business materials, partner technical evaluation

---

## 1. Why memory is the core problem for AI products

AI applications often impress at launch, but after three months of real use two complaints reliably surface — "it remembers all the wrong things," and "it has me confused with someone else." This is not a model-intelligence problem; it is a memory-system design problem.

Most existing memory solutions are write-only: a passing remark and a genuinely important fact are stored with equal weight, and within six months the memory store is choked with noise, dragging answer quality down. This is the most cited pain point of mem0 and similar generic solutions in production (mem0's own GitHub Issue #4573 reports a production case: 10,134 memories of which 97.8% were garbage or duplicates).

When designing the Memex framework, Sentino made this the primary constraint: **the longer a user stays, the better the AI should know them — not the messier it gets.**

---

## 2. Three product values of the Sentino memory framework

### 2.1 The longer it runs, the better the AI knows the user

Sentino's memory system tags every entry with an importance level and uses that level to decide retention:

- **Core information** (identity, family, long-term habits, chronic conditions): permanent
- **Important information** (long-term preferences, ongoing projects, newly formed relationships): 30 days
- **Normal information** (everyday events, near-term plans): 14 days
- **Trivial information** (today's mood, what was just eaten, in-the-moment phrasing): 3 days

Expired entries fade out automatically, while the underlying data remains traceable (recoverable when compliance or audit requires it). When a topic is mentioned repeatedly, the system extends its life automatically — important things stay alive naturally; things no one talks about anymore fade away naturally.

This keeps the memory store lean over time. Sentino's view: **this matches how human memory actually works** — people don't remember everything that happened with equal weight either.

### 2.2 Costs are predictable

mem0 and similar solutions, by default, invoke a large model on every incoming user message to extract memories. As conversation volume grows, token cost scales linearly with message count — the main reason monthly cost runs out of control in high-frequency scenarios such as voice companionship, customer support, and consultation.

Sentino uses **asynchronous batch processing**: user messages are persisted first (millisecond-level return), and once a threshold is reached or a timer fires, a single LLM call processes all new messages together. Under equivalent conditions, processing 19 conversation turns drops from 5 minutes to 15 seconds — a **19× speedup, with token cost reduced to roughly 1/15 of the baseline.**

For the customer this means: ingestion latency stays very low, memory generation happens within predictable batch windows, and monthly cost can be estimated and capped.

### 2.3 Separating "who the user is" from "what we have been through together"

In mem0 and similar systems, memory is a flat list of facts: "user lives in Beijing" and "user took a business trip to Beijing last week" sit side-by-side in the same form. Sentino separates the two at the data layer:

- **User Profile**: the user's current state — an attribute snapshot, updated by overwrite
- **Shared Events**: a timeline of interactions with the user — append-only, with decay

This distinction is critical for AI companion and conversational products. Users implicitly expect two different capabilities:

| Capability | Product experience it powers |
|---|---|
| **Knows who I am** | Personalised recommendations, identity-aligned responses |
| **Remembers what we've been through** | Emotional continuity, a sense of relationship depth |

Mixed together, neither experience comes out well.

---

## 3. Technical direction: why not vector retrieval

This is the deepest technical disagreement between Sentino and mem0, and a key signal for judging the maturity of any memory system.

### 3.1 Inherent problems with vector retrieval

Vector retrieval is "semantic similarity matching" — both memories and queries are converted into mathematical vectors, then ranked by similarity and the top few are returned. This was the mainstream approach for memory systems in 2022–2023. Its problems:

- **It pulls the wrong thing**: "I'm allergic to peanuts" and "I love peanut butter" have very high semantic similarity but opposite meaning. If the AI acts on the wrong memory, health, safety, or compliance risks can follow.
- **Not auditable**: when the AI gives a wrong answer, engineers cannot explain "why this entry was chosen and not that one" — vector space is a black box.
- **Not editable**: when a user asks the system to "forget something," there is no precise location for "that thing" inside the vector store to delete.
- **Not predictable**: because of the approximate nature of vector index search, the same query may return inconsistent results.

### 3.2 Sentino's choice: structured text memory + LLM judgement

Sentino stores memory as structured text and, at retrieval time, lets the LLM read the relevant set and judge what is useful — **rather than asking a vector to "guess" what is relevant.** This means:

- Every memory entry is human-readable, editable, and auditable
- When users ask to modify or forget something, it can be located precisely
- Regulatory or compliance review can trace the basis of every AI decision

### 3.3 Industry validation: Anthropic chose the same path

Anthropic's Claude Code, released in 2025 (widely regarded as one of the most advanced AI coding assistants in the industry), uses the same design — **Markdown text files plus an index, with no vector database.** Anthropic's engineering documentation states the reasoning explicitly: transparent and editable, native Git versioning, natural integration with the file system.

The industry consensus shifted in 2025–2026: as LLM context windows grew and inference cost fell, "letting the model read the relevant set and decide" became more accurate, more explainable, and easier to maintain than "pre-filtering with vector retrieval" in most scenarios.

**Sentino chose this path — same direction as Anthropic.**

---

## 4. Comparison with mem0 (customer view)

| Question the customer cares about | mem0 | Sentino Memex |
|---|---|---|
| Will the memory store still be clean after three months? | Easily bloats; needs manual maintenance | Automatic decay; sharper over time |
| How is monthly token cost controlled? | Scales linearly with conversation volume | Batch processing; monthly cost predictable |
| When the AI gets it wrong, can we find out why? | Vector retrieval is opaque; hard to localise | Text memory is readable and auditable |
| Can a "forget this" request be honoured? | Hard to locate precisely | Locatable down to specific entries |
| Are "who the user is" and "what we've been through" distinguished? | Mixed together | Separated at the data layer |
| Multilingual conversation support? | Depends on configuration | Built-in multilingual memory; unified prompt architecture optimises cache hit rate |

---

## 5. Where it fits — and where it does not

To help customers judge whether Sentino Memex is a fit, the positioning difference from mem0 should be stated honestly:

**Where mem0 wins**: general-purpose Agent memory infrastructure, mature open-source ecosystem, active community, support for many vector database adapters. If the product is a tool-type AI (coding assistant, research assistant, internal knowledge retrieval) with reasonable tolerance for memory imprecision, mem0 is a sensible choice.

**Where Sentino Memex wins**: **products where memory quality directly determines user retention** — AI companionship, voice conversation, customer consultation, relationship-style assistants. These scenarios demand an order of magnitude more from memory precision, cost predictability, and long-term non-decay, and they need trade-offs designed specifically for that.

Sentino is not trying to compete with mem0 in the general-purpose Agent memory space. We focus on scenarios where **memory is the product itself.**

---

# Appendix: Technical Specifications

## A. System architecture

```
Conversation input ──→ Blob ingest (ms-level) ──→ Flush trigger ──→ AI extraction ──→ Storage write
   │                                                                    │
   └─ HTTP API                                                          ├─ User Profile
                                                                        ├─ Memory Event
                                                                        └─ Event Gist (queryable entries)
```

**Deployment form**: Spring Boot service (JDK 17+, Maven 3+)
**Storage backend**: PostgreSQL (primary data) + Redis (cache / locks)
**External dependency**: any LLM service compatible with the OpenAI API (GPT / Claude / Gemini / domestic models all supported)

## B. Data model

| Entity | Purpose | Key fields |
|---|---|---|
| `Blob` | Raw conversation input | userId, projectId, blobData, status, createdAt |
| `UserProfile` | User attribute snapshot | content, attributes (JSONB), updatedAt |
| `UserEvent` | Event record | eventData (event description + profile delta) |
| `UserEventGist` | Smallest retrievable memory unit | gistData, importanceLevel, status, **forgetAt** |

Multi-tenant support: all data is isolated by the composite key `(userId, projectId)`.

## C. Importance levels and TTL

| Level | Retention | Examples |
|---|---|---|
| CORE | Permanent | Identity, family, long-term habits (≥3 months), chronic conditions |
| IMPORTANT | 30 days | Long-term preferences, ongoing projects, newly formed relationships |
| NORMAL (default) | 14 days | Everyday events, weekly plans, general statements |
| TRIVIAL | 3 days | Current mood, what was eaten today, vague phrasing |

---

*This document is maintained by the Sentino team. Feedback and partnership inquiries are welcome.*
