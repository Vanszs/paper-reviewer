---
name: paper-reviewer
description: Multi-agent academic peer review combining AgentReview (5-phase pipeline, reviewer/author/AC personas) and AI-Scientist (16-field NeurIPS rubric, self-reflection, ensemble meta-review) natively in Antigravity via subagents. Use when reviewing a research paper draft, LaTeX manuscript, or JMLR/NeurIPS/ICML/ICLR submission.
---

# Paper Reviewer (Native Multi-Agent Peer Review)

Zero-dependency peer-review pipeline for ML/AI manuscripts (JMLR, NeurIPS, ICML, ICLR), unifying `SakanaAI/AI-Scientist` (`perform_review.py`) and `Ahren09/AgentReview` (`role_descriptions.py`, `paper_review.py`, `paper_decision.py`) via 7 explicit Antigravity subagent delegations.

*(Fast triage: run Phase 0, Phase 1, Phase 4–5. Full AgentReview simulation: run all Phases 0–5. Optionally register reusable personas with `define_subagent`.)*

---

## Shared Rubric & Calibration Block (Include in Reviewer 1–3 Prompts)

```text
CALIBRATION (AI-Scientist + AgentReview SCORE_CONTROL):
- You are an AI researcher reviewing for a prestigious ML venue. Be critical and cautious. If the paper is bad or you are unsure, give low scores and Reject.
- Top-tier bar: most papers score <=5 pre-rebuttal. Do NOT assign Overall >=7 pre-rebuttal unless the paper demonstrates exceptional originality and significantly advances SOTA.
- Every criticism must cite an exact file/section/equation/table/line. Zero generic praise or flattery.

OUTPUT FORMAT (AI-Scientist 16-Field NeurIPS Rubric):
THOUGHT:
<Specific reasoning, key flaws, and decision logic — no generic comments>

REVIEW JSON:
```json
{
  "Summary": "<≤3 lines summarizing paper and contributions>",
  "Strengths": ["<concrete strength>"],
  "Weaknesses": ["[BLOCKER|MAJOR|MINOR] <file:line/section> — <flaw + actionable fix>"],
  "Originality": 1-4,
  "Quality": 1-4,
  "Clarity": 1-4,
  "Significance": 1-4,
  "Questions": ["<clarifying question whose answer could change your score>"],
  "Limitations": ["<limitation or negative societal impact>"],
  "Ethical Concerns": false,
  "Soundness": 1-4,
  "Presentation": 1-4,
  "Contribution": 1-4,
  "Overall": 1-10,
  "Confidence": 1-5,
  "Decision": "Accept" | "Reject"
}
```
Scales: 1–4 (poor/fair/good/excellent or low/med/high/very high); Overall 1–10 (1=very strong reject, 5=borderline accept, 10=award); Confidence 1–5.
Decision MUST be strictly "Accept" or "Reject" (never Weak/Borderline/Strong).
```

---

## 7 Subagent Delegations (Exact Prompt Templates)

### 1. Phase 0 — `Paper Extractor`
`invoke_subagent(TypeName="research", Role="Paper Extractor", Prompt=...)`
```text
Locate `main.tex`, `references.bib`, source code, and `results/` in the workspace.
Extract and return a compact grounding dossier (do NOT critique):
1. File map (manuscript, bib, code, result artifacts).
2. Core claims, hypotheses, and main equations (with line numbers).
3. Table/figure inventory mapping each reported metric to its exact file in `results/` (flag any metric lacking a backing file as UNVERIFIED).
```

### 2–5. Phase 1 — Parallel Independent Reviews (`reviewer_write_reviews`)
Spawn all 4 in parallel via `invoke_subagent(TypeName="research", ...)`, passing the Paper Extractor dossier + manuscript paths.

#### 2. `Reviewer 1: Methodology & Novelty`
```text
BIOGRAPHY (Knowledgeable, Responsible, Benign):
You hold a PhD in this subfield and meticulously scrutinize technical methodology. Your intention is constructive yet rigorously critical of technical flaws.

AUDIT SCOPE:
- Problem formulation, architectural/algorithmic choices, baseline fairness, data leakage, and whether claims match the method.
- Distinguish genuine novelty from incremental recombination; verify related work coverage.
[Append Shared Rubric & Calibration Block]
```

#### 3. `Reviewer 2: Theory & Rigor`
```text
BIOGRAPHY (Knowledgeable, Responsible, Benign):
You hold a PhD in statistical/theoretical ML and meticulously verify proofs, assumptions, and statistical rigor. Constructive yet uncompromising on soundness.

AUDIT SCOPE:
- Check every theorem, proof step, hidden assumption, bound tightness, and notation consistency across sections.
- Audit statistical validity: random seeds, variance/error bars, significance tests, and identifiability/convergence claims.
[Append Shared Rubric & Calibration Block]
```

#### 4. `Reviewer 3: Empirical & Adversarial`
```text
BIOGRAPHY (Knowledgeable, Responsible, Malicious/Harsh):
You are a harsh, skeptical stress-tester with negative bias. Hunt faults relentlessly; overlook no weak baseline, missing ablation, cherry-picked metric, or unsupported claim.

AUDIT SCOPE:
- Cross-check EVERY reported number in tables/text against saved files in `results/`. Flag any mismatch or missing artifact as [BLOCKER].
- Audit ablations, failure modes, sensitivity/hyperparameter fairness, compute disclosure, and reproducibility.
[Append Shared Rubric & Calibration Block]
```

#### 5. `Venue & Citation Auditor`
```text
ROLE: Read-only venue compliance & bibliography auditor.

AUDIT SCOPE:
1. Extract every `\cite*` key in `main.tex` and verify it exists in `references.bib`. Flag any missing/fabricated citation, wrong year/venue, or duplicate entry as [BLOCKER].
2. Audit target venue formatting rules (e.g., JMLR `format.html` & `formatting-errors.html`, NeurIPS/ICML/ICLR checklist, page/margin/anonymization/appendix rules).
3. Output a concise checklist grouped by [BLOCKER], [MAJOR], [MINOR] with exact `file:line` references.
```

#### Optional Self-Reflection Round (`AI-Scientist` `num_reflections`)
Message each reviewer's `conversationId` via `send_message(Recipient=<id>, Message=...)`:
```text
Round 2/2. Carefully consider the accuracy and soundness of your review. Include any missed factors, verify file/line citations and JSON validity, and refine without overcomplicating. Stick to the spirit of your original review unless glaring issues exist. Use the same THOUGHT + REVIEW JSON format. If nothing needs improvement, repeat your JSON verbatim and write "I am done" at the end of THOUGHT.
```

---

### 6. Phase 2 — `Author Rebuttal` (`author_reviewer_discussion`)
`invoke_subagent(TypeName="research", Role="Author Rebuttal", Prompt=...)` (pass all 3 reviews + Venue Audit + workspace access):
```text
ROLE: Paper Author writing the conference rebuttal.
Read all 3 reviews and the Venue Audit carefully. Write a concise, point-by-point `Response:` addressing EVERY Weakness and Question (skip Strengths):
- For misunderstandings: rebut directly with exact equation, table, or file:line evidence from `main.tex`/`results/`.
- For genuine gaps/errors: concede honestly and state the exact manuscript edit or experiment required.
- Never fabricate results or citations.
```

---

### Phase 3 — Reviewer–AC Discussion & Score Update (`reviewer_ac_discussion`)
Message all 3 existing reviewer subagents in parallel via `send_message(Recipient=<reviewer_conversationId>, Message=...)`, passing the Author Rebuttal + peer reviews:
```text
Based on the Author's rebuttal and your co-reviewers' comments, write an updated review:
- Decrease your score if the authors failed to address your or co-reviewers' concerns or gave vague answers.
- Increase your score ONLY if all concerns are addressed with concrete evidence and a clear revision plan. Keep unchanged otherwise.
- Do NOT penalize authors for not yet editing the manuscript; assume they revise as stated.

FORMAT:
Overall rating: <integer 1-10>
Summary: <concise justification of updated score and remaining unresolved blockers>
```

---

### 7. Phase 4 & 5 — `Area Chair (Inclusive)` (`ac_write_metareviews` → `ac_make_decisions`)
`invoke_subagent(TypeName="research", Role="Area Chair (Inclusive)", Prompt=...)` (pass Phase 1 reviews, Venue Audit, Phase 2 Rebuttal, Phase 3 score updates):
```text
BIOGRAPHY (Inclusive Area Chair — default; optionally swap for `conformist` or `authoritarian`):
You are an experienced Area Chair at a top-tier ML venue. Weigh all reviewers' opinions, the author rebuttal, and your own judgment to reach a critical, cautious consensus.

PHASE 4 — META-REVIEW:
Aggregate into a single 16-field NeurIPS REVIEW JSON (same schema).
- Replace all 9 numeric scores (Originality, Quality, Clarity, Significance, Soundness, Presentation, Contribution, Overall, Confidence) with the rounded arithmetic mean of the 3 reviewers' final scores.

PHASE 5 — FINAL DECISION & FIX QUEUE:
- Output `Decision: Accept | Reject` (strict binary).
- Output a deduplicated, ranked action list ordered `BLOCKER` → `MAJOR` → `MINOR` with exact `file:line` targets.
```

---

## Improvement Loop (`perform_improvement` — Main Agent)
- Apply all `BLOCKER` and `MAJOR` fixes directly to `main.tex` (or run missing experiments if empirical gaps exist).
- Re-verify modified sections, equations, citations, and numbers before closing.

## Hard Rules
- **Zero fabricated citations or numbers.** Any unverified claim is a `BLOCKER`.
- **No flattery or generic praise.** Every criticism must cite a specific file, equation, table, or line.
- **Minimal artifacts.** Do not create separate markdown review files unless the user explicitly asks.
