---
name: dialecticate
description: Run a structured dialectic (thesis → antithesis → synthesis) between two Dialectician-role workers on a topic, question, or design problem given as $ARGUMENTS. Use when the user wants a dialectic on a topic, to debate both sides of a decision, to stress-test an idea through adversarial argument, or to reach a position that survives the strongest objections. Invoke via /code-dialectic:dialecticate <topic or question>.
---

# Dialecticate

Run a rigorous two-worker dialectic on the topic or problem the user provides as `$ARGUMENTS`. You (the conductor) orchestrate; the two `dialectician` workers do the reasoning. Never author the arguments yourself.

## 1. Frame
From `$ARGUMENTS`, write one precise, falsifiable **proposition** the dialectic will test. If the topic is vague, or it's a code/architecture question tied to a specific repo, ask the user one clarifying question (scope + which project) — do not guess.
Pick a project to spawn into:
- A code/design question about a specific repo → spawn into that project (workers may read its code).
- A purely conceptual question → spawn into an existing scratch project, or create one (e.g. via `create_project`) if none exists yet.

## 2. Spawn two dialecticians
Spawn **two** workers:
- `spawn_instance({ project: <chosen>, mode: "bypassPermissions", model: "code-dialectic/dialectician" })`
- No worktree — these are reasoning workers; nothing to mutate or isolate. `bypassPermissions` because they only produce prose; nothing to approve.
- Capture both `sessionId`s. Label them **Thesis** (A) and **Antithesis** (B).

## 3. Dialectic rules (send to BOTH in their first prompt)
> Dialectic rules:
> 1. **Steelman, never strawman** — restate your opponent's strongest form before critiquing.
> 2. **Concede valid points** — mark concessions explicitly; keep a running "Agreed" list.
> 3. **Charity in interpretation, rigor in evaluation** — fair to the other side, hard on the arguments.
> 4. **Number your claims** (C1, C2…) so they can be referenced across rounds.
> 5. **End every turn with a STANCE SUMMARY** (≤5 lines): current position, concessions, open disputes.
> 6. **Stay on the proposition** — no goalpost-shifting.
> 7. **No premature consensus** — surface real disagreement before synthesis.
>
> Proposition: <the proposition>

Then assign stance:
- **Thesis** (R1): "You are the Thesis advocate. Make the strongest possible case *for* the proposition."
- **Antithesis** (R1): "You are the Antithesis advocate. Attack the thesis — flaws, counterexamples, hidden assumptions — and make the strongest case *against* the proposition."

Drive every turn via dispatch-and-wake: `send_prompt({sessionId, subscribe:true})`, then **end your turn**. Never hold your turn open while a worker runs. Resubscribe on each wake while rounds remain.

## 4. Round cadence (relay)
Default **3 rounds** (offer the user more/fewer). Each round is a pair of turns. Relay outputs between the workers: paste the opponent's latest turn verbatim into the other's next prompt, prefixed `Opponent's last turn:`, and re-state the rules block (rules drift if omitted).
- R1: Thesis → wake → relay → Antithesis → wake.
- R2: relay → Thesis refines (concede + defend) → wake → relay → Antithesis counters → wake.
- R3: same. Each round must escalate depth — new arguments, not repetition.
Before each relay, confirm the worker ended with a STANCE SUMMARY; if not, ask it to add one before continuing.

## 5. Termination
Stop after N rounds **or** when both stance summaries show no remaining substantive dispute. Do not run extra rounds past convergence.

## 6. Synthesis round
In the final turn, ask **both** workers (independently, same prompt) to write a **Synthesis** — not a compromise:
- What survived scrutiny (the refined position).
- What was conceded, and why.
- The residual disagreement / crux, if any.
Then **you (the conductor) compose the final synthesis** from the two drafts: prefer the position that survived the strongest objections; surface the crux honestly if they don't converge. Do not split the difference; do not manufacture agreement.

## 7. Output to the user
Present: (1) the proposition; (2) a round-by-round summary (thesis, antithesis, key concessions — a few lines each); (3) the final synthesis (refined position + residual crux). End with the sentinel `DIALECTIC_COMPLETE`. Then retire both workers (`kill_instance`).
