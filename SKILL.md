---
name: paper-reviewer
description: Multi-agent academic peer review combining AgentReview (5-phase pipeline, reviewer/author/AC personas) and AI-Scientist (16-field NeurIPS rubric, self-reflection, ensemble meta-review) natively in Antigravity via subagents. Use when reviewing a research paper draft, LaTeX manuscript, or JMLR/NeurIPS/ICML/ICLR submission.
---

# Paper Reviewer (Native Multi-Agent Peer Review)

Zero-dependency peer-review pipeline for ML/AI manuscripts (JMLR, NeurIPS, ICML, ICLR), faithfully unifying `SakanaAI/AI-Scientist` (`perform_review.py`) and `Ahren09/AgentReview` (5-phase pipeline) using native Antigravity subagents.

## Pipeline & Subagent Mapping

### Phase 0: Grounding (`Paper Extractor`)
Spawn `invoke_subagent(TypeName="research", Role="Paper Extractor")`:
- Locate `main.tex`, `references.bib`, code, and `results/`; extract structured paper summary, claims, equations, and table/result paths for downstream subagents.

### Phase 1: Independent Reviews + Self-Reflection (`reviewer_write_reviews`)
Spawn 4 independent read-only subagents in parallel via `invoke_subagent(TypeName="research", ...)` (strictly 1 role per subagent):
1. **`Role="Reviewer 1: Methodology & Novelty"`** — Persona: *Knowledgeable, Responsible, Benign*. Audits problem formulation, architectural choices, baseline fairness, data leakage, and claim-method alignment.
2. **`Role="Reviewer 2: Theory & Rigor"`** — Persona: *Knowledgeable, Responsible, Benign*. Audits theorems, proofs, assumptions, notation consistency, and statistical validity (seeds, error bars, significance tests).
3. **`Role="Reviewer 3: Empirical & Adversarial"`** — Persona: *Knowledgeable, Responsible, Malicious/Harsh* (adversarial stress-tester). Cross-checks EVERY table/text number against `results/`, audits ablations, failure modes, and compute disclosure.
4. **`Role="Venue & Citation Auditor"`** — Read-only compliance check. Verifies every `\cite*` key exists in `references.bib`, checks zero fabricated citations, and audits venue formatting rules (e.g., JMLR `format.html` & `formatting-errors.html`).

*(For fast triage, skip Phase 2–3; for full AgentReview simulation, run all 5 phases. To reuse custom personas across papers, optionally register them with `define_subagent`.)*

**Calibration (`AI-Scientist` + `AgentReview`):**
- Be critical and cautious: if a paper is flawed or you are unsure, give low scores and `Reject`.
- Top-tier bar: assign `Overall <= 5` pre-rebuttal unless the work demonstrates exceptional originality and rigor.

**Required Output — AI-Scientist 16-Field NeurIPS Rubric:**
Each reviewer outputs `THOUGHT:` (concise reasoning) followed by `REVIEW JSON:` containing all 16 fields in order:
- `Summary` (string, ≤3 lines)
- `Strengths` (list of concrete items)
- `Weaknesses` (list of items with `file:line/section` + severity `BLOCKER` | `MAJOR` | `MINOR`)
- `Originality` (1–4), `Quality` (1–4), `Clarity` (1–4), `Significance` (1–4)
- `Questions` (list of clarifying questions for authors)
- `Limitations` (list of limitations & negative societal impacts)
- `Ethical Concerns` (boolean)
- `Soundness` (1–4), `Presentation` (1–4), `Contribution` (1–4)
- `Overall` (1–10), `Confidence` (1–5)
- `Decision` (`Accept` | `Reject` only — no Weak/Borderline variants)

**Self-Reflection (`AI-Scientist` `num_reflections`):**
Each reviewer performs 1 self-reflection pass checking accuracy, soundness, and JSON validity before finalizing (or message its `conversationId` via `send_message` for an extra round until `"I am done"`).

### Phase 2: Author Rebuttal (`author_reviewer_discussion`)
Spawn `invoke_subagent(TypeName="research", Role="Author Rebuttal")` with all 3 reviews + access to `main.tex` and `results/`:
- Respond point-by-point to every Weakness and Question (skip Strengths).
- Distinguish genuine manuscript/empirical gaps (admit and propose concrete fix) from reviewer misunderstandings (cite exact equation/table/file).

### Phase 3: Reviewer–AC Discussion & Score Update (`reviewer_ac_discussion`)
Message all 3 existing reviewer subagents in parallel via `send_message(Recipient=<reviewer_conversationId>, ...)`, passing the Author rebuttal and peer reviews:
- Each reviewer outputs an updated `Overall` score (1–10) + concise justification.
- Decrease score if concerns remain unaddressed/vague; increase only if all concerns are resolved with concrete evidence.

### Phase 4 & 5: Area Chair Meta-Review & Final Decision (`ac_write_metareviews` → `ac_make_decisions`)
Spawn `invoke_subagent(TypeName="research", Role="Area Chair (Inclusive)")` with Phase 1 reviews, Phase 2 rebuttal, and Phase 3 updates. *(AC persona: `inclusive` default; optionally `conformist` or `authoritarian`.)*
- **Phase 4 (Meta-Review):** Synthesize consensus in the same 16-field NeurIPS JSON format. Replace all 9 numeric scores (`Originality`, `Quality`, `Clarity`, `Significance`, `Soundness`, `Presentation`, `Contribution`, `Overall`, `Confidence`) with the rounded mean of the 3 reviewers' final scores.
- **Phase 5 (Final Decision):** Issue final `Decision` (`Accept` | `Reject`) and ranked action list (`BLOCKER` → `MAJOR` → `MINOR`).

### Improvement Loop (`perform_improvement` — Main Agent)
- Apply all `BLOCKER` and `MAJOR` fixes directly to `main.tex` (or run missing experiments if empirical gaps exist).
- Re-verify modified sections and numbers before closing.

## Hard Rules
- **Zero fabricated citations or numbers.** Flag any unverified claim as `BLOCKER`.
- **No flattery or generic praise.** Every criticism must cite a specific equation, table, or line.
- **Minimal artifacts.** Do not create separate markdown review files unless the user explicitly asks.
