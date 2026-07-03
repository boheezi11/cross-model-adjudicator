---
name: cross-model-refinement
description: Run a structured cross-model adversarial refinement loop that pressure-tests and iteratively improves PM deliverables — case study answers, PRDs, slide narratives, concept one-pagers, positioning, brainstorms, product-market-fit arguments — using Claude as generator/adjudicator and ChatGPT (via a paste bridge or API) as external critic, until both models score the work 9.5/10 or higher against a fixed rubric AND no P0 issues remain. Use this skill whenever the user asks to "refine", "red-team", "pressure-test", "get ChatGPT feedback on", "run the loop on", or "converge" a deliverable; whenever they paste feedback from ChatGPT or another model; or whenever they say a draft is "done" and want to verify it would survive an interview panel or exec review.
---

# Cross-Model Refinement Loop

A structured critique-and-converge workflow that turns "paste ChatGPT feedback and see what sticks" into a repeatable agent with roles, a rubric, an audit trail, and a defensible stopping rule.

## Why this exists

The pattern this automates was proven manually on the Front Copilot take-home: eight pieces of external (ChatGPT) feedback arrived, and the winning behavior was **triage, not obedience** — take wholesale (an unprovable "fastest-growing" claim), take with a tweak, or push back with reasoning (a "describe an example" note answered better by a prototype screenshot). The same pattern held on Stampli and BILL: fabricated-metric catches, verified-vs-inferred labeling, honest competitive concessions. This skill encodes that adjudication step so critique makes the work sharper without eroding its spine.

## The four roles

**1. Generator (Claude).** Produces or ingests the draft. On later rounds, applies the approved ledger items and nothing else.

**2. Critic (ChatGPT, highest available model).** Red-teams the draft against `rubric.md`. Not "give feedback" — find the claims a sharp interviewer would challenge, the fabricated or unverifiable metrics, segmentation drift, jargon, and structural weaknesses. Delivered via the **paste bridge** (see below) or API.

**3. Adjudicator (Claude).** Triages every critique item into **TAKE / TWEAK / REJECT**, each with a one-line reason. A REJECT must cite a specific rubric principle it would violate (e.g., "would break one-segment-one-pain" or "asks us to claim something unverified"). This is the anti-capitulation mechanism: the loop is not allowed to accept feedback just because it arrived.

**4. Dual Judge (both models, independently).** Scores the revised draft on every rubric dimension with cited evidence from the draft itself, and **must list its top 3 remaining weaknesses regardless of score**. A 9.5 with no nameable substantive weakness is legitimate convergence; a 9.5 alongside a P0-grade weakness is score inflation, and the loop continues.

## Convergence rule (the stopping condition)

The loop is DONE only when ALL of the following hold:

1. Both models score the draft **≥ 9.5 overall** on the rubric
2. **Zero open P0 items** in the ledger (P0 = fabricated/unverifiable load-bearing claim, internal contradiction, or a flaw that fails the deliverable's core ask)
3. Each judge's "top 3 remaining weaknesses" are all P2-level (polish), not substantive
4. **The human approves.** Adjudicator triages first; the human gives final approval on convergence and on any REJECT of a P0-severity critique. The human can override any triage decision at any time.

Hard cap: **5 rounds.** If not converged by round 5, stop and present the human with the open ledger and both judges' dissents — an honest "here is where the models disagree" beats a forced fake consensus.

## The loop, step by step

### Round 0 — Setup
1. Ingest the draft (pasted text, file, or built in-conversation).
2. Confirm the deliverable type and its **core ask** (what a "pass" means: the brief's questions, the interview panel's criteria, etc.). Write this at the top of the ledger — every P0 judgment traces back to it.
3. Read `rubric.md`. If the deliverable type suggests dimension weight changes (e.g., a PRD weights Framework Rigor higher than Narrative), propose them to the human before starting.
4. Create the round ledger from `template.md`.

### Each round — Critique → Adjudicate → Approve → Apply → Judge
1. **Generate the critic prompt.** Build a single self-contained block from `assets/critic-prompt.md`: rubric + severity definitions + the current full draft. Instruct the human: "Paste this into ChatGPT (highest model, extended thinking if available) and paste the full response back."
2. **Parse the critique.** Normalize into ledger items: `#, severity (P0/P1/P2), quoted text, the challenge, proposed fix`.
3. **Adjudicate.** Claude triages each item TAKE / TWEAK / REJECT with reasons. Claude also adds its **own** critique items here — the external critic does not have a monopoly on finding flaws.
4. **Human checkpoint.** Present the triaged ledger compactly. Human approves, overrides, or adds items. Do not apply anything before this.
5. **Apply.** Revise the draft with approved items only. Log each change against its ledger item number.
6. **Dual judge.** Generate the judge prompt from `assets/judge-prompt.md` for ChatGPT (paste bridge); Claude scores independently in the same format *before* seeing GPT's scores (write Claude's scores into the ledger first, then request the GPT paste — this ordering prevents anchoring).
7. **Check convergence.** If met → present final draft + full ledger for human sign-off. If not → next round.

### Context reset (critical)
Each round starts fresh with only: (a) the current draft, (b) the open-ledger items, (c) the core-ask statement. Never carry the full history of prior critiques, dead ends, and superseded drafts into the critic or judge prompts — per the context-engineering principle, bounded context per role. Resolved ledger items are archived in the log file, not re-fed to the models.

## Severity definitions

- **P0** — Fabricated, unverifiable, or falsifiable load-bearing claim; internal contradiction (e.g., segmentation switches dimension mid-argument); failure against the deliverable's core ask. *Blocks convergence.*
- **P1** — Weakens the argument or credibility but doesn't break it: soft evidence where hard exists, buried lede, mechanism-identical "alternatives," missing counter-metric, hedged recommendation.
- **P2** — Polish: wording, slide economy, cadence, formatting, AI-sounding phrasing.

## Anti-inflation guardrails (read before judging)

LLM judges drift toward polite convergence by round 2–3 and inflate scores to end loops. Counters built into this skill:

1. **Evidence-anchored scoring** — every dimension score must quote the draft text that earns it. No quote, no score.
2. **Mandatory weaknesses at any score** — judges list top 3 remaining weaknesses even at 9.5+. Substantive weaknesses invalidate the score.
3. **Independent, ordered judging** — Claude commits its scores to the ledger before the GPT judge paste is requested.
4. **One-directional severity** — an item's severity can be raised in later rounds but never quietly lowered; downgrades require an explicit human approval line in the ledger.
5. **Round cap with honest dissent** — a non-converged round 5 outputs disagreement, not a manufactured 9.5.

## Paste bridge protocol (v1, no API key)

The GPT side runs through the user's ChatGPT account manually. To keep this cheap in human effort:
- Exactly **two pastes per round** (one critique, one judge). Never ask the user to relay follow-up questions to GPT; if the critique is ambiguous, the adjudicator resolves it or marks the item "needs human read."
- Critic and judge prompts must be fully self-contained — GPT sees no other context, so include the rubric, severity definitions, and full current draft every time.
- If the draft is too long for one paste, split by section and instruct GPT to hold critique until the final part ("PART 1 of N — reply only 'received'").

## API upgrade path (v2)

The prompts in `assets/` are the durable artifact; swapping the paste bridge for API calls changes plumbing only. When the user gets an OpenAI API key: critic and judge calls go to the highest available GPT model via `/v1/chat/completions` with the same prompt text; expected cost is roughly $1–2 per deliverable per full loop at flagship pricing (verify current rates at platform pricing pages before quoting). Claude-side roles can run in Claude Code or via the Anthropic API. The ledger format is unchanged.

## Files

- `rubric.md` — the quality rubric with dimensions, weights, and scoring anchors. Read at Round 0, embed in every critic/judge prompt.
- `template.md` — the round ledger template. Instantiate at Round 0, maintain every round.
- `assets/critic-prompt.md` — the self-contained critique prompt for the paste bridge or API.
- `assets/judge-prompt.md` — the self-contained scoring prompt, used verbatim by both judges.

## Pitfalls

1. **Accepting all critique** — the adjudicator exists precisely because external feedback is sometimes wrong for this deliverable. The Front #7 case (a good general note, beaten by a better specific move) is the canonical example.
2. **Rejecting to protect ego** — every REJECT must cite a rubric principle. "I like it the way it is" is not a citation.
3. **Score-watching** — the number is the least informative output. The open-P0 count and the judges' named weaknesses are the real signal.
4. **Context bleed** — feeding prior rounds' full critique history back to the models degrades quality and inflates cost. Reset every round.
5. **Skipping the human checkpoint** — the human approves before apply, and signs off on convergence. The agent proposes; the human disposes.
6. **Treating 9.5 as the goal** — the goal is a deliverable that survives a sharp panel. The 9.5 is a proxy; when proxy and goal conflict, the goal wins and the ledger records why.
