---
name: fleet-orchestration
description: "Delegation policy for a price/performance agent stack built on three separate ~$20/month plans — Claude (main loop + sub-agents), ChatGPT Plus (Codex lanes), and a Google account (Gemini Flash via Antigravity `agy`). The main loop owns architecture, judgment, AND per-lane routing (model + effort); delegates execute exactly what it stamps. Codex: gpt-6.1-sol for every routed lane with effort scaled by difficulty, gpt-6-astra only on explicit user request, defensive security lanes on sol `high` read-only, upgraded to gpt-daybreak-blue-latest only when the account has Daybreak access. Gemini Flash: generous, high-volume grunt tier. Claude Sonnet 5.5 via agy (separate small quota group): fallback executor when Codex is out of quota. Claude sub-agents share the main loop's quota, so they get an explicit, cheapest-capable model. Load whenever spawning sub-agents, Codex or Gemini lanes, or planning any delegation."
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
| Google (agy Claude/GPT group) | same Google account, separate quota | Claude Sonnet 5.5 lanes through `agy` | fallback executor when Codex is out — small, spend deliberately |

Core law: **route each piece of work to the cheapest pool that can do it
right.** The main loop's tokens buy judgment, not throughput.

## The hard rules

1. **Claude sub-agents are not free.** They draw on the same plan as the main
   loop. Always set the model explicitly on every Agent tool / workflow
   `agent()` call — never omit-and-inherit, or you silently fan out your most
   expensive model. Pick the cheapest Claude model that can carry the job
   (`haiku` < `sonnet` < `opus`; Claude 5 family, 5.5 generation as of
   2026-10), and prefer the main loop doing small work hands-on over
   reflexive delegation. Fable 5.1 is not on this ladder until its quota
   cost against the plan has been measured.
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
  - `gpt-6.1-sol` `medium` — routine execution against a complete spec:
    refactors along an existing pattern, tests against a defined contract,
    read-heavy investigation.
  - `gpt-6.1-sol` `high` — hard or precision lanes with a complete spec.
  - `gpt-6.1-sol` `xhigh` — a single, explicitly heavy lane. Never a
    fleet-wide reflex; heavy effort burns Plus quota too. (`max`/`ultra`
    exist on sol but are opt-in only.)
  - `gpt-6.1-sol` (2026-09-29) replaced `gpt-5.6-sol` as the lane model:
    near-astra agentic coding per OpenAI, and on Plus it draws from the
    GPT-6 Sol usage group, which stretches further than 5.6-sol's. If it is
    unavailable, fall back to `gpt-6-sol`, then `gpt-5.6-sol`. Luna models are
    not routed; cheap bulk work belongs to Gemini Flash.
  - **`gpt-6-astra` is never auto-routed.** It is frontier-tier but drains a
    Plus plan fast. Use it only when the user explicitly asks.
  - **Defensive security lane — `sol` by default, Daybreak Blue when
    available.** Any lane whose deliverable is a security judgment on systems
    the operator owns or is authorised to test:
    security review / audit of a diff or module, vulnerability triage,
    threat modelling, secret and credential-leak sweeps, dependency/CVE impact
    analysis, auth / sandbox / permission boundary review, hardening plans,
    security-focused incident or log analysis, authorised CTF work.
    - Model: `gpt-6.1-sol`. Upgrade to `gpt-daybreak-blue-latest` (Daybreak
      Blue, OpenAI's approval-gated Trusted Access for Cyber tier: same
      frontier model with cyber safeguards relaxed) only when the account is
      known to have Daybreak access. Access requires approval, an eligible
      plan, Advanced Account Security and two FIDO2 hardware keys, and can be
      revoked; don't assume it. If a Daybreak lane fails with a
      model-unavailable / access error, re-fire the same brief on `sol` at the
      same effort instead of dropping the lane.
    - Daybreak is an upgrade, not a requirement: it mainly cuts refusals on
      legitimate defensive work. Plain `sol` handles code review, triage and
      threat modelling of your own code fine.
    - Effort: `medium` for triage and single-file checks, `high` default for
      audits, `xhigh` for one explicitly deep audit. `max`/`ultra` only on
      explicit request.
    - Audit / triage lanes run `--sandbox read-only`. A security *fix* lane may
      write in its own worktree against a spec derived from verified findings.
    - Findings are claims: the main loop verifies each against the live code
      before anything is fixed or reported. The gate review of a
      security-critical seam still stays in the main loop.
    - Offensive work (exploitation of third-party systems, evasion, malware)
      is not a routing case for any lane.
    - Daybreak quota burn relative to `sol` is not yet measured; disclose the
      lane count as usual and don't fan out more than a few Daybreak lanes at
      once.
  - Effort buys more thinking, not a higher ceiling. The single most
    contract-sensitive or security-critical seam stays in the main loop
    (security lanes feed it evidence; they don't replace its gate review).
  - Lane cost scales with repo size, not change difficulty: ~30–50K tokens
    per write lane in a small, self-contained repo, ~100–150K for a medium
    lane in a real project (measured 30K / 37K / 142K on 2026-09-27; no write
    lane goes much below ~30K). Put lanes × the upper end of the matching
    range in the disclosure sentence, and keep minute-sized jobs in the main
    loop.
  - Codex models are obsessive instruction followers: capable, but they
    *execute* rather than improvise. Lane quality is bounded by spec quality —
    invest main-loop tokens in the brief, not in doing the lane yourself.
- **Claude Sonnet via `agy` (`gemini-fleet`, stamped `claude-sonnet-5-5-*`) —
  the Codex fallback.** Antigravity bills Claude/GPT models from a quota group
  separate from Gemini Flash, and none of it touches the Claude plan. When
  Codex hits its usage limit mid-task, re-route spec'd execution lanes here
  instead of pulling them into the main loop: `claude-sonnet-5-5-high` for
  write lanes, `-medium` for routine ones. Same lane-brief contract as a
  Codex lane; the main loop still verifies and gates the result.
  - The group is small: each Antigravity window is roughly 500K fresh input
    tokens per 5 hours and about two such windows per week (measured
    2026-10-09 on `claude-opus-5-5-low`; Sonnet stretches further, consumption
    is proportional to token cost). One medium write lane on Sonnet `high`
    took ~144K fresh tokens. A multi-turn lane re-sends its context every
    turn, so a single long lane can empty a window.
  - `claude-opus-5-5-*` through `agy` is emergency-only: a fresh call costs
    ~2% of the weekly window (~21.5K input overhead per call), so roughly
    45–50 short calls a week. Not a routing target.
  - Write lanes need `--dangerously-skip-permissions`, and Claude Code's
    auto-permission check refuses to launch that from the main loop. Hand the
    exact command to the operator to run with the `!` prefix, then pick up the
    result from its log and worktree.
  - Not for bulk grunt work (that is Gemini Flash) and not a reason to skip
    reporting the Codex limit to the operator first.
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
(effort by difficulty), or agy Sonnet 5.5 when Codex is out of quota; defensive security analysis → Codex sol `high` read-only (Daybreak Blue if the account has it); loose exploration → cheapest capable Claude
sub-agent; judgment / specs / synthesis / gate review → the main loop.**

## The difficulty axis

| Work | Route | Pool |
|---|---|---|
| Uniform, shallow, high-volume | Gemini `flash-low` / `flash-medium` | Google |
| Routine, spec'd, parallel | Codex `sol` `medium` | ChatGPT |
| Hard / precision, spec'd | Codex `sol` `high` | ChatGPT |
| One explicitly heavy lane | Codex `sol` `xhigh` | ChatGPT |
| Spec'd execution while Codex is out of quota | agy `claude-sonnet-5-5-high` (`-medium` routine), operator launches write lanes | Google (Claude/GPT group) |
| Defensive security: audit, vuln triage, threat model, secret/CVE sweep | Codex `sol` `high` (`medium` triage), read-only; `gpt-daybreak-blue-latest` if available | ChatGPT |
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
  For an exhausted Codex pool that next pool is agy Sonnet 5.5 (above).
- The operator can override any of this per session; absent that, this policy
  stands.
