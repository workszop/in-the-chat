# PRD — "Inside the Chat" Educational Platform (v2.0)

**Product:** Inside the Chat — an interactive learning platform explaining how AI chat systems work
**Document owner:** Andrzej Jankowski (Quantica Lab)
**Status:** Draft for review
**Date:** 23 July 2026
**Version:** 1.0
**Predecessor:** "Inside the Chat" v1 — single-page static simulator (App vs Model, Tokenization, Embeddings, Generation, Recap)

---

## 1. Executive summary

v1 proved the core pedagogical concept: learners understand LLMs best when they *manipulate* the pipeline rather than read about it. However, v1 is fully scripted — every probability, token split, and chat reply is hand-authored. v2 expands the product into a full educational application with ten learning modules covering the complete lifecycle of an AI chat interaction, where **every demonstration that can be live, is live**: real tokenizers, real embeddings, real next-token probabilities, and real model calls executed through an LLM API behind a secure proxy.

The product ships with two runtime modes per module:

- **Simulation Mode** (default, free, offline-capable) — deterministic scripted demos, suitable for classrooms without connectivity or budget.
- **Live Mode** (API-powered) — the same interaction executed against real models, so learners can verify that the simplified story matches reality.

Primary markets: postgraduate AI programs, corporate AI training (executive / employee / technical tiers), and self-directed learners. The platform is bilingual (PL/EN) from day one.

---

## 2. Background and problem statement

Most people now use AI chatbots daily, yet mental models remain wrong in ways that cause real harm: users treat chatbots as databases (expecting factual lookup), as beings with memory (expecting persistence that doesn't exist), or as oracles (not verifying output). Existing explainers are either too technical (research visualizations of transformer internals) or too shallow (metaphor-only articles). There is a gap for a **rigorous but accessible, interactive, bilingual** learning product that:

1. Separates the *application layer* from the *model* — the single most misunderstood distinction.
2. Lets learners *do* each pipeline stage themselves, with real systems, not just watch animations.
3. Is honest about its own simplifications (every module carries a "What we simplified" disclosure).
4. Works both as self-study material and as a live instructor tool in a lecture hall.

---

## 3. Goals and non-goals

### 3.1 Goals

| # | Goal | Measure |
|---|------|---------|
| G1 | Learners can correctly explain the app/model split, tokenization, embeddings, and autoregressive generation after one session | ≥80% pass rate on post-module knowledge checks |
| G2 | Maximize interactivity — every core concept has at least one hands-on element | 100% of modules contain ≥1 interactive exercise; ≥60% offer a Live Mode |
| G3 | Usable in real classrooms | Instructor mode adopted in ≥3 course editions in first semester |
| G4 | Sustainable API cost | ≤ $0.15 average Live Mode cost per learner-hour (see §10) |
| G5 | Full PL/EN parity | All UI, content, quizzes and prompts localized |

### 3.2 Non-goals (v2)

- Not a general chatbot product or a prompt-engineering productivity tool.
- Not a math course: no backpropagation derivations, no matrix calculus (linked as external further reading).
- No user-generated public content; no social features.
- No fine-tuning of custom models; training is explained, not performed (except the RLHF annotation game, which simulates the data-labeling step only).
- No mobile native apps (responsive web only).

---

## 4. Target users and personas

| Persona | Context | Needs | Depth |
|---|---|---|---|
| **P1 Postgraduate student** (Biznes.AI / AI-law programs) | Guided cohort learning, mixed technical background | Rigorous but jargon-free; certifiable progress | Medium–high |
| **P2 Corporate learner** (employee tier) | 2–4 h company training | Fast intuition, safety/limits awareness, practical takeaways | Low–medium |
| **P3 Technical learner** (developers, IT) | Deep-dive workshops, self-study | Real API payloads, parameters, internals, code-adjacent detail | High |
| **P4 Instructor** | Lectures and workshops | Presenter view, classroom session control, cost caps, offline fallback | n/a |
| **P5 Self-directed learner** | Public web | Free tier, no account required for Simulation Mode | Variable |

Audience-to-module depth mapping is defined per module via three content layers: **Intuition** (always visible), **Mechanics** (expandable), **Under the hood** (technical layer with payloads, parameters, references).

---

## 5. Product principles

1. **Show, then tell.** Interaction first; explanation appears alongside, never as a wall of text before the demo.
2. **Factually correct simplification.** Every simplification is disclosed. No metaphor may contradict the underlying mechanism.
3. **Simulation ↔ Live parity.** Both modes present the same UI; Live Mode replaces scripted data with real model output, teaching learners that the simulation was honest.
4. **The learner is the scientist.** Modules are structured as experiments: predict → run → compare → explain.
5. **Cost-aware by design.** Live calls are cached, capped and always degradable to Simulation Mode.

---

## 6. Product architecture overview

```
┌───────────────────────────── Frontend (SPA, PL/EN) ─────────────────────────────┐
│  Learning Modules M1–M10   ·   Guided Paths   ·   Quizzes   ·   Glossary        │
│  Mode switch: [Simulation] [Live]        Instructor / Presenter view            │
└───────────────▲─────────────────────────────────────────────▲───────────────────┘
                │ scripted data (bundled JSON)                │ HTTPS (session token)
                │                                             │
        ┌───────┴────────┐                        ┌───────────┴───────────────┐
        │ Local engines   │                        │  Backend proxy ("Relay")  │
        │ · demo tokenizer│                        │  · provider abstraction   │
        │ · in-browser    │                        │  · key vault (no keys in  │
        │   tokenizers    │                        │    the browser, ever)     │
        │   (WASM)        │                        │  · caching (exact+semantic)│
        │ · small local   │                        │  · rate & budget limiter  │
        │   model for     │                        │  · moderation & logging   │
        │   attention viz │                        └───────┬───────────┬───────┘
        └────────────────┘                                 │           │
                                              Anthropic Claude API   OpenAI API
                                              (chat, token counting) (logprobs,
                                                                      embeddings)
```

**Provider strategy (verified against current provider capabilities):**

| Capability | Provider / mechanism | Rationale |
|---|---|---|
| Conversational modules (M1, M6, M7, M8) | Anthropic Claude (Messages API) | Quality of explanation-style output; streaming; system-prompt control |
| Exact token counting for a given model | Anthropic Token Counting endpoint; client-side WASM tokenizers (tiktoken/HF) for other vocabularies | Real counts without generation cost |
| **Next-token probability distributions** (M5) | **OpenAI API with `logprobs` + `top_logprobs`** | Anthropic's Messages API does not expose logprobs; OpenAI returns up to 20 top candidates per position — exactly the data M5 visualizes |
| Embedding vectors (M3, M9) | OpenAI embeddings (or Voyage AI as alternative) | Anthropic offers no first-party embeddings endpoint |
| Attention visualization (M4) | Small open model executed in-browser (e.g. GPT-2-class via transformers.js/ONNX) | Frontier APIs do not expose attention weights; only a locally-run open model can show real ones |
| Quiz feedback / LLM-as-judge | Anthropic Claude, small/cheap tier | Structured JSON grading of free-text answers |

This multi-provider design is itself teachable and is surfaced in the "Under the hood" layer of each module.

---

## 7. Functional requirements — learning modules

Each module follows the same internal structure: **(a) Hook** (30-second interactive teaser), **(b) Core interactive**, **(c) Explanation layers** (Intuition / Mechanics / Under the hood), **(d) Experiment tasks** (2–4 guided challenges), **(e) Knowledge check** (3–5 questions), **(f) "What we simplified"** disclosure.

Requirement priority: **[P0]** = MVP-blocking, **[P1]** = fast-follow, **[P2]** = later phase.

### M1 — Anatomy of a Chat (App vs Model) [P0]

Carried over from v1 and upgraded to Live Mode.

- **M1.1 [P0]** Split view: rendered chat left, X-ray right. In Live Mode the X-ray shows the *actual JSON request body* sent to the proxy (system prompt, messages array, parameters) and the raw streamed response events.
- **M1.2 [P0]** Editable system prompt: learner rewrites the system prompt (e.g. "Answer only in pirate Polish") and immediately observes behavior change — proving where "personality" lives.
- **M1.3 [P0]** Free-typed user messages in Live Mode (moderated; see §9.4), preset scripted messages in Simulation Mode.
- **M1.4 [P1]** "Forgetting demo": a toggle drops older turns from the request; the model visibly loses knowledge of the user's name — statelessness made tangible.
- **M1.5 [P1]** Live token counter per request using the token-counting endpoint; running cost estimate displayed per turn.
- **M1.6 [P2]** Provider switcher: send the same transcript to two different models side by side; compare replies.

### M2 — Tokenization Lab [P0]

- **M2.1 [P0]** Free-text input tokenized *client-side with real production tokenizers* (WASM builds of at least two vocabularies, e.g. GPT-class BPE and an open-model tokenizer), rendered as colored chips with real token IDs.
- **M2.2 [P0]** Tokenizer comparison view: same text, 2–3 tokenizers side by side, counts compared.
- **M2.3 [P0]** Language fairness explorer: preloaded parallel sentences (EN/PL/DE/JP) showing token-count differences and the cost implication for non-English languages.
- **M2.4 [P1]** Cost calculator: paste any text → tokens → price at current per-model rates (rates maintained in a config file, not hardcoded).
- **M2.5 [P1]** "Strawberry challenge": interactive exercise where the learner predicts, then verifies in Live Mode, whether a model can count letters — with the tokenization-based explanation of failure.
- **M2.6 [P2]** Build-your-own-BPE mini-game: learner merges character pairs by frequency on a toy corpus, discovering how vocabularies are learned.

### M3 — Embedding Space [P0]

- **M3.1 [P0]** Live Mode: learner types any words/phrases → real embedding vectors fetched via proxy → projected to 2D (PCA/UMAP computed client-side) → plotted alongside a preloaded reference cloud of ~200 common words.
- **M3.2 [P0]** Click-two similarity meter (cosine on the *full* vectors, not the 2D projection — displayed with the note that the map is a lossy shadow).
- **M3.3 [P1]** Analogy sandbox: vector arithmetic (king − man + woman) resolved by nearest-neighbor search over the reference set; results shown honestly, including when the trick fails.
- **M3.4 [P1]** Semantic search mini-demo: 30 preloaded sentences; learner types a query; ranking by embedding similarity vs. keyword match shown side by side — bridging to M9 (RAG).
- **M3.5 [P2]** Contextual embeddings demo: "bank" in two sentences produces different vectors (requires local model or sentence-level embeddings; scoped in design phase).

### M4 — Attention & The Transformer [P1] *(new)*

- **M4.1 [P1]** In-browser small open model runs on learner-provided sentences; real attention heatmaps rendered (token-to-token grid + arc diagram), with head/layer selectors.
- **M4.2 [P1]** Guided walkthrough: a scrollytelling sequence following one sentence through embedding → attention → feed-forward → output logits, each stage animated from the model's actual tensors (dimensionality-reduced).
- **M4.3 [P2]** "Ambiguity probe": preset sentences with pronoun references ("The trophy didn't fit in the suitcase because *it* was too big"); learners inspect which tokens attend to "it".
- **M4.4 [P0-content]** Even before the interactive ships, the module page exists with the explanation layers and a scripted animation, so the pipeline narrative has no gap.

### M5 — Generation Lab (Probabilities & Sampling) [P0]

The flagship upgrade from v1.

- **M5.1 [P0]** Live Mode: any prompt → real top-k next-token distribution retrieved via logprobs → animated probability bars. Step / autoplay generation where each appended token comes from the real distribution.
- **M5.2 [P0]** Temperature, top-p and top-k controls wired to real API parameters; greedy toggle; side-by-side "run twice" button demonstrating (non-)determinism.
- **M5.3 [P0]** Knowledge-as-probability demo: factual prompts ("The capital of Poland is") showing near-certain distributions vs. open prompts showing flat ones — the conceptual bridge to hallucination (M7).
- **M5.4 [P1]** "Beat the model" game: the learner guesses the next token before revealing the distribution; scored across 10 rounds; leaderboard within a classroom session.
- **M5.5 [P1]** Perplexity intuition: color the tokens of a pasted text by how surprised the model was at each position.
- **M5.6 [P2]** Sampling-strategy visual comparison: same prompt continued 5× at three temperatures, displayed as a branching tree.

### M6 — Context Window & Memory [P1] *(new)*

- **M6.1 [P1]** Interactive context budget bar: system prompt + history + user message + reply visualized as segments filling a fixed-size window; learner keeps chatting until overflow forces trimming, watching what falls out.
- **M6.2 [P1]** Memory strategies explainer: raw history vs. summarized history vs. external memory (file/notes) — each strategy selectable in a live chat, with the X-ray showing what actually gets sent.
- **M6.3 [P2]** Needle-in-a-haystack mini-experiment: hide a fact at a chosen position inside filler text; test retrieval in Live Mode.

### M7 — Hallucinations & Limits Lab [P1] *(new)*

- **M7.1 [P1]** Confidence ≠ correctness: curated prompts where models produce fluent falsehoods; learner rates plausibility before seeing the verdict; running personal "detector score".
- **M7.2 [P1]** Verification workflow trainer: a model answer with claims; learner marks which claims need checking; guided comparison against provided sources.
- **M7.3 [P1]** Knowledge-cutoff and "cannot know" demos (today's weather, private facts); why the correct behavior is refusal or tool use.
- **M7.4 [P2]** Prompt-injection safety demo (sandboxed, education-only): a "poisoned document" changes an assistant's behavior in a contained simulation; ties to the AI-law audience's governance interests.

### M8 — Prompt Engineering Sandbox [P1] *(new)*

- **M8.1 [P1]** A/B prompt bench: two prompt variants against the same task run side by side in Live Mode; diffed outputs.
- **M8.2 [P1]** Technique cards (role, few-shot, chain-of-thought, output constraints, structured JSON): each card is a one-click applied transformation on the learner's current prompt, with before/after.
- **M8.3 [P2]** LLM-as-judge scoring of learner prompts against rubrics for 5 standard exercises, with structured feedback.

### M9 — RAG: Giving the Model Your Documents [P2] *(new)*

- **M9.1 [P2]** Upload or paste a document (client-side only in Simulation; via proxy in Live) → visualized chunking → embeddings (reusing M3 machinery) → live retrieval for a question → answer with per-chunk citations, each pipeline stage inspectable.
- **M9.2 [P2]** Retrieval failure gallery: wrong chunk size, ambiguous query, contradictory sources — each as a runnable experiment.

### M10 — How Models Are Made (Training & Alignment) [P1] *(new)*

Primarily narrative with two interactives; no actual training performed.

- **M10.1 [P1]** Scale explorer: interactive log-scale visualization of dataset size, parameter counts and compute across model generations.
- **M10.2 [P1]** RLHF annotation game: learner ranks pairs of model responses like a human annotator; after 10 rankings, sees how their preferences would shape a reward signal — demystifying "alignment".
- **M10.3 [P2]** Base-vs-instruct demo: the same prompt sent to a base-style completion setup vs. a chat-tuned model (implementable via a raw-completion open model in-browser), showing why instruction tuning matters.

---

## 8. Cross-cutting product features

### 8.1 Learning paths & progression [P0]

- Three curated paths mapped to personas: **Essentials** (P2 corporate, ~90 min: M1, M2, M5, M7), **Full Course** (P1 students, ~6 h: M1–M8, M10), **Technical Track** (P3, all modules with "Under the hood" layers unlocked by default).
- Linear-with-freedom navigation: recommended order, everything accessible.
- Progress persisted locally by default; account-based sync is [P1] and optional (see §9.5).

### 8.2 Knowledge checks & feedback [P0]

- Per-module quizzes (multiple choice + one free-text "explain it in your own words").
- [P1] Free-text answers graded by LLM-as-judge (structured rubric, JSON output), returning targeted feedback, never just a score. Fallback: self-assessment against a model answer when Live Mode is off.
- [P1] Pre/post assessment pair for the Full Course path to measure learning gain (metric G1).
- [P2] Completion certificate (PDF) for the Full Course path.

### 8.3 Instructor / Classroom mode [P1]

- Presenter view: large-type rendering of any interactive, keyboard-driven, hides explanation text.
- Classroom sessions: instructor creates a session code; students join without accounts; instructor's Live Mode budget applies with a hard per-session cap; class-aggregated results for the M5.4 game and quizzes displayed on the presenter screen.
- One-click "freeze to simulation" switch for connectivity failures mid-lecture.

### 8.4 Content infrastructure [P0]

- Full i18n (PL/EN) for UI, module content, quiz items, and the prompts sent to models (system prompts localized so Live Mode answers arrive in the learner's language).
- Glossary: every term (token, logit, temperature, context window…) is a hoverable definition; glossary page auto-compiled.
- "What we simplified" disclosure block mandatory in every module (editorial rule, enforced in content schema).
- Content stored as structured data (headless content files), not hardcoded in components, to keep translation and review manageable.

### 8.5 Accessibility [P0]

- WCAG 2.1 AA: full keyboard operation of all interactives, ARIA live-regions for streamed text, color-independent encodings (probability bars carry numeric labels; token colors are decorative), reduced-motion mode disabling autoplay animations.

---

## 9. LLM integration — technical requirements

### 9.1 Proxy ("Relay") [P0]

- All model calls go through a backend proxy; API keys never reach the browser.
- Provider abstraction with per-module routing table (see §6) and per-provider fallback chains.
- Streaming pass-through (SSE) for chat modules.
- Request shaping: the proxy owns final prompts; the client sends structured intents (module ID, exercise ID, learner inputs), which limits abuse surface and enables caching.

### 9.2 Caching & cost control [P0]

- Exact-match cache on (module, exercise, normalized input, parameters) — classroom usage is highly repetitive; expected hit rate >60% for preset exercises.
- Per-session token budget with visible meter in the UI; when exhausted, graceful drop to Simulation Mode with a clear explanation (which is itself a teachable moment about inference cost).
- Global daily spend circuit breaker with alerting.
- Cheap-tier models by default; module config may pin specific models where pedagogy requires (e.g. logprobs support).

### 9.3 Rate limiting & abuse prevention [P0]

- Per-session and per-IP rate limits; CAPTCHA-free friction via progressive delays.
- Free-text inputs length-capped (e.g. 500 chars) and screened by a moderation step before reaching generation models.
- The proxy strips/ignores any attempt to override module system prompts, except inside M1.2 and M8, where system-prompt editing is the exercise — there, outputs are additionally moderated.

### 9.4 Safety & content policy [P0]

- Moderation on both learner input and model output in Live Mode; blocked content shows an age-appropriate educational notice.
- The platform must be usable by minors in supervised settings: no data collection from learners in classroom sessions, conservative moderation thresholds, no open-ended chat outside module contexts.

### 9.5 Privacy & compliance [P0]

- GDPR/RODO: Simulation Mode collects nothing; Live Mode processes learner text transiently (no retention beyond cache TTL; cache keys hashed). Optional accounts store only e-mail, locale and progress.
- EU AI Act transparency: clear labeling that learners are interacting with AI systems in Live Mode; provider list and data-flow description on a public transparency page. (Also a differentiator for the AI-law audience.)
- EU-region hosting for the proxy and any stored data.

---

## 10. Cost model (planning estimate)

Assumptions for a 90-minute Essentials session in Live Mode, cheap-tier models, cache cold:

| Module | Live calls / learner | Est. tokens / call (in+out) | Est. cost / learner |
|---|---|---|---|
| M1 chat turns | 6 | 1,200 | ~$0.02 |
| M2 token counting | 10 | counting endpoint / client-side | ~$0.00 |
| M5 logprob steps | 25 | 300 | ~$0.03 |
| M7 exercises | 4 | 1,500 | ~$0.02 |
| Quiz feedback | 4 | 800 | ~$0.01 |
| **Total (cold cache)** | | | **~$0.08** |

With expected classroom cache hit rates, marginal cost per learner drops further — comfortably inside the G4 target of $0.15/learner-hour. Exact figures to be re-based on current provider price sheets at implementation time (prices change frequently; maintained in config).

---

## 11. Technical stack (recommendation)

- **Frontend:** React + TypeScript SPA; design system carried over from v1 (minimalist enterprise style, CSS custom properties, inline SVG, Inter). D3 for the embedding map and probability visualizations; WASM tokenizers; transformers.js/ONNX Runtime Web for the in-browser model (M4, M10.3), lazy-loaded only when those modules open.
- **Backend:** lightweight Node/TypeScript service (containerized; EU region) implementing the Relay: provider SDKs, Redis for cache + rate limiting, structured logging without payload retention.
- **Content:** MDX/JSON content files, PL/EN, in-repo with review workflow.
- **Testing:** provider-mocked integration tests for every Live exercise; visual regression on interactives; a "pedagogy test suite" — scripted assertions that Simulation and Live modes produce structurally identical UI states.

---

## 12. Success metrics

| Metric | Target (first 6 months) |
|---|---|
| Module completion rate (started → knowledge check passed) | ≥ 55% |
| Learning gain (post-test vs pre-test, Full Course) | +30 pp average |
| Live Mode adoption among connected users | ≥ 40% of sessions |
| Instructor sessions run | ≥ 25 |
| Avg. Live cost per learner-hour | ≤ $0.15 |
| Learner satisfaction (post-course, 1–5) | ≥ 4.3 |
| Accessibility audit | AA pass, zero blockers |

Instrumentation: privacy-light product analytics (no session recording; event counts only), quiz outcomes, cost telemetry per module.

---

## 13. Release plan

| Phase | Scope | Duration (est.) |
|---|---|---|
| **Phase 1 — MVP** | Platform shell, PL/EN i18n, Simulation Mode for M1–M5 + M10 narrative, Live Mode for M1, M2, M3, M5; Relay with caching/limits/moderation; Essentials path; quizzes (MCQ) | 10–12 weeks |
| **Phase 2 — Classroom** | Instructor mode + sessions; M6, M7, M8; M4 interactive (in-browser model); LLM-graded free-text; accounts + progress sync; certificates groundwork | +8–10 weeks |
| **Phase 3 — Depth** | M9 (RAG), M10 interactives, M4/M5 advanced items, certificate, needle-in-haystack, base-vs-instruct | +8 weeks |

Milestone gates: Phase 1 exit requires G4 cost validation in a pilot classroom (n≥20) and the pedagogy test suite green.

---

## 14. Risks and mitigations

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| Provider API changes (logprobs availability, pricing) | M5 Live degraded | Medium | Provider abstraction; Simulation parity means no module ever breaks fully; config-driven model routing |
| In-browser model too heavy for classroom laptops | M4 unusable | Medium | Quantized small model, lazy load, server-side rendering fallback of precomputed attention for preset sentences |
| Live costs exceed budget in open free tier | Financial | Medium | Session budgets, cache, cheap tiers, Live Mode behind lightweight sign-in for anonymous web users |
| Free-text inputs produce unsafe outputs in classrooms | Reputational | Low–Med | Dual moderation, length caps, module-scoped prompting, conservative defaults for classroom sessions |
| Translation drift between PL and EN content | Quality | Medium | Single-source content schema; translation review checklist per release |
| Scope creep (10 modules) | Delivery | High | Strict P0/P1/P2 gating; Phase 1 ships with 6 modules only |

---

## 15. Open questions

1. Account model for individual learners: local-only progress vs. optional accounts at MVP — decision needed before Phase 1 design freeze.
2. Monetization: free public tier + paid classroom licenses vs. fully free with institutional sponsorship — affects budget-cap UX.
3. LTI/SCORM integration for university LMS platforms — Phase 3 candidate; validate demand with two partner programs first.
4. Which second and third tokenizers to bundle (licensing and WASM size trade-offs).
5. Whether M7.4 (prompt-injection demo) needs legal review for the AI-law audience before publication.

---

## 16. Acceptance criteria — MVP definition of done

1. A learner with no account completes the Essentials path in Simulation Mode fully offline after first load.
2. Switching M1/M2/M3/M5 to Live Mode changes only the data source, not the interface; the X-ray in M1 shows the genuine request/response payloads.
3. M5 Live displays a real top-k next-token distribution for arbitrary user prompts and generates step-by-step using real sampling with learner-controlled temperature.
4. M3 Live plots real embedding vectors for learner-typed words with correct cosine similarity on full vectors.
5. No API key is retrievable from the client; budget exhaustion degrades gracefully to Simulation Mode with an explanatory message.
6. Every shipped module contains its "What we simplified" disclosure in both languages.
7. All interactives are operable by keyboard alone and pass automated AA checks.

---

## Appendix A — Module × persona relevance matrix

| Module | P1 Student | P2 Corporate | P3 Technical |
|---|---|---|---|
| M1 App vs Model | ●●● | ●●● | ●●● |
| M2 Tokenization | ●●● | ●●○ | ●●● |
| M3 Embeddings | ●●● | ●○○ | ●●● |
| M4 Attention | ●●○ | ○○○ | ●●● |
| M5 Generation | ●●● | ●●● | ●●● |
| M6 Context & Memory | ●●● | ●●○ | ●●● |
| M7 Hallucinations | ●●● | ●●● | ●●○ |
| M8 Prompting | ●●○ | ●●● | ●●● |
| M9 RAG | ●●○ | ○○○ | ●●● |
| M10 Training & Alignment | ●●● | ●○○ | ●●○ |

## Appendix B — Simulation/Live parity contract (excerpt)

For every Live-capable exercise, the module config must declare: `intent_id`, input schema, scripted dataset reference (Simulation), provider route + parameters (Live), cache key recipe, moderation profile, and the UI state schema both modes must satisfy. The pedagogy test suite validates that scripted and live payloads render through identical component states.
