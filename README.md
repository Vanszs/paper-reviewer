# Paper Reviewer

A zero-dependency, multi-agent academic peer-review skill (`SKILL.md`) for ML/AI research manuscripts targeting **NeurIPS**, **ICML**, **ICLR**, and **JMLR** standards.

It runs natively via coding-agent subagents with strict **1-subagent-per-role** isolation—combining structured rubric evaluation, adversarial empirical auditing, author rebuttal, reviewer score updates, and Area Chair meta-review synthesis.

## Credits & References

This skill faithfully adapts and unifies two foundational peer-review frameworks into a single portable `SKILL.md`:

- **[SakanaAI/AI-Scientist](https://github.com/SakanaAI/AI-Scientist)** — 16-field NeurIPS review rubric (`perform_review.py`), strict score calibration, self-reflection passes (`num_reflections`), and ensemble meta-review score aggregation.
- **[Ahren09/AgentReview](https://github.com/Ahren09/AgentReview)** — 5-phase peer-review simulation pipeline (`reviewer_write_reviews` → `author_reviewer_discussion` → `reviewer_ac_discussion` → `ac_write_metareviews` → `ac_make_decisions`), distinct reviewer personas (benign vs. harsh/adversarial), author rebuttal, and Area Chair personas (`inclusive`, `conformist`, `authoritarian`).

## Architecture (1-Subagent-Per-Role)

Every role runs in its own isolated subagent context to prevent cross-contamination between reviewers, authors, and the Area Chair:

| Phase | Role | Mode | Responsibility |
| :--- | :--- | :--- | :--- |
| **Phase 0** | **Paper Extractor** | Read-only subagent | Locates `main.tex`, `references.bib`, code, and `results/`; extracts structured claims, equations, and table/result paths for downstream agents. |
| **Phase 1** | **Reviewer 1: Methodology & Novelty** | Read-only subagent *(Benign)* | Audits problem formulation, architectural choices, baseline fairness, data leakage, and claim–method alignment. |
| **Phase 1** | **Reviewer 2: Theory & Rigor** | Read-only subagent *(Benign)* | Audits theorems, proofs, assumptions, notation consistency, and statistical validity (seeds, error bars, significance tests). |
| **Phase 1** | **Reviewer 3: Empirical & Adversarial** | Read-only subagent *(Malicious/Harsh)* | Stress-tests every table/text number against `results/`, audits ablations, failure modes, and compute disclosure. |
| **Phase 1** | **Venue & Citation Auditor** | Read-only subagent | Verifies every `\cite*` key against `references.bib`, flags fabricated citations, and checks venue formatting rules (e.g., JMLR). |
| **Phase 2** | **Author Rebuttal** | Read-only subagent | Responds point-by-point to every Weakness and Question, separating genuine gaps from misunderstandings with exact file/equation citations. |
| **Phase 3** | **Reviewers 1–3 (Discussion)** | Existing subagents (`send_message`) | Reviews the rebuttal and peer critiques; issues updated `Overall` scores (1–10) with concise justification. |
| **Phase 4–5** | **Area Chair (Inclusive)** | Read-only subagent | Synthesizes consensus in the 16-field NeurIPS JSON format (with mean reviewer scores), issues `Accept` / `Reject`, and outputs a ranked `BLOCKER` → `MAJOR` → `MINOR` action list. |
| **Post-Review** | **Main Agent** | Read/Write | Executes the improvement loop: applies `BLOCKER` and `MAJOR` fixes directly to `main.tex`, runs missing experiments if needed, and re-verifies numbers. |

## Installation

Install via [skills.sh](https://skills.sh):

```bash
npx skills add Vanszs/paper-reviewer
```

Or install globally across supported agents:

```bash
npx skills add Vanszs/paper-reviewer -g
```

## Usage

Once installed, trigger the skill in any research manuscript repository (containing `main.tex`, `references.bib`, and/or `results/`):

- **Full 5-phase simulation:** *"Review this manuscript using the `paper-reviewer` skill targeting NeurIPS / JMLR standards."*
- **Fast triage (Phases 0–1 + AC):** *"Run a fast `paper-reviewer` triage on `main.tex` to find all BLOCKER and MAJOR issues."*

### Hard Rules Enforced
- **Zero fabricated citations or numbers** — any unverified claim is flagged as `BLOCKER`.
- **No generic praise** — every critique cites a specific file, line, equation, or table.
- **Minimal artifacts** — avoids littering your workspace with unsolicited markdown files unless explicitly requested.
