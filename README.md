# cross-model-adjudicator

A cross-model adversarial refinement loop for PM deliverables — case study answers, PRDs, concept one-pagers, positioning docs, brainstorms. Claude generates and adjudicates; ChatGPT (highest available model) red-teams; both judge independently against a fixed rubric. The loop converges only when **both models score ≥ 9.5/10, zero P0 issues remain, and a human approves**.

Born from a manual workflow used on real PM interview take-homes: external model feedback was never accepted wholesale — every item was triaged into **take / tweak / reject-with-reasoning** against a fixed set of principles (verified-vs-inferred labeling, one-segment-one-pain focus, plain spoken language, user pain before business outcome). This repo encodes that triage step as the core of the agent.

## How it works

```
Draft ──► Critic (GPT) ──► Adjudicator (Claude) ──► Human approval ──► Apply
                                                                        │
        ◄────────────── next round (context reset) ◄── Dual Judge ◄─────┘
```

Four roles, bounded context per role:

1. **Generator** (Claude) — produces or ingests the draft; on later rounds applies approved ledger items only.
2. **Critic** (GPT) — red-teams against the rubric: fabricated/unverifiable claims, segmentation drift, business-problems-dressed-as-user-pain, structural flaws. Every issue must quote the draft and carry a severity (P0/P1/P2).
3. **Adjudicator** (Claude) — triages every critique into TAKE / TWEAK / REJECT. A REJECT must cite a rubric principle. This is the anti-capitulation mechanism.
4. **Dual Judge** (both, independently) — evidence-anchored scores per rubric dimension, plus a mandatory "top 3 remaining weaknesses" at any score, which defeats score inflation.

**Stopping rule:** both judges ≥ 9.5 overall (no dimension < 9) AND zero open P0s AND remaining weaknesses are polish-only AND human sign-off. Hard cap of 5 rounds — a non-converged round 5 outputs honest dissent, not forced consensus.

**Context reset:** each round sees only the current draft + open ledger items + the core-ask statement. Prior rounds' critique history is archived, never re-fed.

## Running it

**v1 — paste bridge (no API key needed).** Install `cross-model-refinement/` as a Claude skill (or drop the files into a Claude Project). Claude orchestrates; the GPT side is exactly two pastes per round into your ChatGPT app using the self-contained prompts in `assets/`.

**v2 — API (planned).** The prompts are plumbing-agnostic. With an OpenAI API key, the two pastes become API calls (~$1–2 per full loop at flagship pricing); the ledger and rubric are unchanged.

## Repo layout

```
cross-model-refinement/
├── SKILL.md                 # the loop: roles, rounds, convergence rule, guardrails
├── rubric.md                # 7 weighted dimensions with scoring anchors
├── template.md              # the round ledger (audit trail)
└── assets/
    ├── critic-prompt.md     # self-contained red-team prompt (paste bridge / API)
    └── judge-prompt.md      # self-contained scoring prompt, used verbatim by both judges
cross-model-refinement.skill # packaged skill, installable in Claude
```

## Design notes

- **Judges inflate scores to end loops.** Counters: evidence-quoted scoring (no quote, no score), mandatory named weaknesses at any score, independent ordered judging (Claude commits before seeing GPT), one-directional severity (downgrades require explicit human approval).
- **The 9.5 is a proxy.** The goal is a deliverable that survives a sharp panel; when proxy and goal conflict, the goal wins and the ledger records why.
