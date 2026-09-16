---
name: fleet-orchestration
description: "Delegation policy for a price/performance agent stack built on three separate ~$20/month plans — Claude (main loop + sub-agents), ChatGPT Plus (Codex lanes), and a Google account (Gemini Flash via Antigravity `agy`). The main loop owns architecture, judgment, AND per-lane routing (model + effort); delegates execute exactly what it stamps. Codex: gpt-5.6-sol for every routed lane with effort scaled by difficulty, gpt-6-astra only on explicit user request. Gemini Flash: generous, high-volume grunt tier. Claude sub-agents share the main loop's quota, so they get an explicit, cheapest-capable model. Load whenever spawning sub-agents, Codex or Gemini lanes, or planning any delegation."
---

# Orchestration & delegation policy ($20-plan stack)

A routing policy for an agent stack where **no single subscription is big
enough to do everything**, but three cheap ones together are. The unit of cost
is not a token price — it's **which plan's quota a piece of work burns**:

| Pool | Plan | Used by | Nature |
|---|---|---|---|
| Claude | ~$20 Claude plan | main loop **and** Claude sub-agents | scarcest — spend on judgment |
| ChatGPT | ~$20 ChatGPT Plus | Codex lanes (`codex-fleet`) | precise executor — spend on spec'd work |
| Google | personal Google account | Gemini Flash lanes (`gemini-fleet`) | high-volume grunt pool — spend freely |

Core law: **route each piece of work to the cheapest pool that can do it
right.** The main loop's tokens buy judgment, not throughput.

## The hard rules

1. **Claude sub-agents are not free.** They draw on the same plan as the main
   loop. Always set the model explicitly on every Agent tool / workflow
   `agent()` call — never omit-and-inherit, or you silently fan out your most
   expensive model. Pick the cheapest Claude model that can carry the job
   (`haiku` < `sonnet` < `opus`), and prefer the main loop doing small work
   hands-on over reflexive delegation.
2. **Model/effort per lane is a routing decision, and routing decisions belong
   to the main loop — never to the executor skill.** `codex-fleet` and
   `gemini-fleet` do not pick their own model or effort; they run whatever the
   main loop stamps on the lane spec. A human or the main loop must always be
   able to answer "why did this lane get this model" by pointing at the spec.
3. **The main loop keeps the big picture.** Architecture, specs,
   contract-sensitive design, subtle state machines, integration and conflict
   resolution, final synthesis, gate review, judgment calls. Mechanical,
   scoped, parallelizable work gets delegated to the other pools.

## Choosing the delegate

- **Gemini Flash via `agy` (`gemini-fleet`) — the grunt tier, used
  generously.** Uniform, shallow, per-item work: one check repeated across 50
  files, mechanical transforms, classification/labelling, bulk scans,
  first-pass triage. When shallow work could go to a Claude sub-agent, Codex,
  or Gemini, lean Gemini — it's the pool least worth hoarding. Default
  `gemini-3.8-flash-low`, `flash-medium` freely when items need light
  reasoning; never `gemini-3.1-pro-*` from this tier. Flash's failure mode is
  confident plausible wrongness: demand structured output, count rows against
  the input, spot-check a sample. Batch tiny items (per-call overhead is
  ~13K tokens), fan out when it buys speed. **No personal data** — lane content
  leaves your machine.
- **Codex (`codex-fleet`) — `sol` for every routed lane; effort carries the
  difficulty.**
  - `gpt-5.6-sol` `medium` — routine execution against a complete spec:
    refactors along an existing pattern, tests against a defined contract,
    read-heavy investigation.
  - `gpt-5.6-sol` `high` — hard or precision lanes with a complete spec.
  - `gpt-5.6-sol` `xhigh` — a single, explicitly heavy lane. Never a
    fleet-wide reflex; heavy effort burns Plus quota too. (`max`/`ultra` need
    astra.)
  - **`gpt-6-astra` is never auto-routed.** It is frontier-tier but drains a
    Plus plan fast. Use it only when the user explicitly asks.
  - Effort buys more thinking, not a higher ceiling. The single most
    contract-sensitive or security-critical seam stays in the main loop.
  - Codex models are obsessive instruction followers: capable, but they
    *execute* rather than improvise. Lane quality is bounded by spec quality —
    invest main-loop tokens in the brief, not in doing the lane yourself.
- **Claude sub-agents — exploration that needs Claude-level judgment.**
  Codebase mapping, research sweeps, reviews, verification passes: work where
  the brief is loose and the deliverable is understanding, not a diff. Use the
  cheapest model that copes with the ambiguity, and only when Gemini (too
  shallow) or Codex (needs a full spec) don't fit.
- **Deep external model (`gptpro` + `gptpro-handoff`) — optional.** A slow,
  non-agentic chat-UI reasoning model for one hard, well-framed audit or design
  question. Which model you get depends on your ChatGPT plan; the workflow
  works with the strongest thinking model your plan offers.

Rule of thumb: **bulk mechanical → Gemini; spec'd execution → Codex sol
(effort by difficulty); loose exploration → cheapest capable Claude
sub-agent; judgment / specs / synthesis / gate review → the main loop.**

## The difficulty axis

| Work | Route | Pool |
|---|---|---|
| Uniform, shallow, high-volume | Gemini `flash-low` / `flash-medium` | Google |
| Routine, spec'd, parallel | Codex `sol` `medium` | ChatGPT |
| Hard / precision, spec'd | Codex `sol` `high` | ChatGPT |
| One explicitly heavy lane | Codex `sol` `xhigh` | ChatGPT |
| Loose exploration needing judgment | Claude sub-agent, explicit cheap model | Claude |
| Most critical seam, gate review, synthesis | main loop | Claude |
| Anything on astra | only on explicit user request | ChatGPT |

When assigning fleet lanes, stamp `{model, effort}` per lane in the spec so the
launch is mechanical and no routing decision happens at spawn time. Before
firing more than one lane at once, say in one sentence how many lanes and which
model/effort each gets — disclosure, not a request for approval. Every pool is
a finite plan quota that a silent fan-out can drain in minutes.

## Exceptions

- If the operator explicitly names a model for a scoped task, honor it for that
  task only, then return to this policy.
- If one pool is exhausted, say so and re-route to the next-cheapest pool that
  can do the work correctly — don't silently push everything onto the main loop.
- The operator can override any of this per session; absent that, this policy
  stands.
