---
name: gemini-fleet
description: Standalone Antigravity CLI (`agy`) runner + fleet orchestrator for GRUNT work. Gemini Flash is the disposable-worker tier of the stack — cheap, high-volume, uniform, mechanical jobs that neither Claude sub-agents (exploration) nor Codex (precision execution) should be spending tokens on: repeating one simple check across 50 files, mechanical transforms, cheap classification/labelling, bulk shallow scans. ALWAYS EXECUTES, never just describes. Runs headless via `agy -p "<prompt>" --model <model>` against the user's logged-in Google account (NO API key). Model/effort per lane are NOT this skill's call — the calling agent (see `fleet-orchestration`) stamps `{model}` on each lane; this skill executes it. Default when nothing is stamped: `gemini-3.8-flash-low`. State lane count and each lane's model in one sentence BEFORE firing more than one lane (disclosure, not a gate). This tier is meant to be used GENEROUSLY: route shallow work here readily and fan out freely. Triggers on: "use gemini", "run agy", "antigravity", "gemini flash", "grunt lane", "bulk check these files", "classify/label this list", "same mechanical edit across N files", "spawn a gemini fleet".
---

# Gemini Fleet — Grunt-Tier Action Runner (`agy`)

> Drives the **Antigravity CLI** (`agy`), Google's official terminal agent and the
> successor to the old Gemini CLI (which was shut down for personal accounts on
> 2026-06-18). Runs entirely against the user's **interactive Google login** —
> **no `ANTHROPIC_API_KEY`, no `GEMINI_API_KEY`, no `GOOGLE_API_KEY`.** Same
> posture as `codex-fleet` using a ChatGPT-account login.
>
> Architecturally this file is a sibling of `codex-fleet/SKILL.md`: same
> background-first invocation, same worktree isolation, same staggered spawn,
> same lane-brief contract, same pre-fleet disclosure rule. Only the executable
> and the tier's *job* differ.

## The niche: soldier / grunt agent

This is the **fourth tier** of the stack. It is not a smarter Codex and not a
cheaper Claude sub-agent. It does the work the other tiers are wasted on:

| Tier | Role | Shape of work |
|---|---|---|
| Main loop (Claude plan — scarcest pool) | judgment, specs, architecture, integration | one hard thing, deeply |
| **Claude sub-agents** (same pool as main loop) | **exploration / context gathering** | loose brief, deliverable is *understanding* |
| **Codex (`sol`; `astra` only on user request)** | **precision execution** | complete spec, a *diff* that must be right |
| **Gemini Flash (`agy`) — this skill** | **grunt / disposable worker** | cheap, high-volume, uniform, mechanical, per-item |

**Usage posture: generous.** In this stack Gemini is the high-volume pool,
not a resource to hoard. When shallow work could go to a Claude sub-agent, Codex
`sol`, or here, lean here. Don't hold back on lanes or on `flash-medium` out of
quota anxiety — the correctness guardrails below still apply, the scarcity
posture does not.

Route a job here when these hold:

- The task is **uniform** — the same instruction applied N times over N items.
- Each item is **shallow** — no cross-file reasoning, no architecture, no
  contract sensitivity, no "figure out what this system does."
- The output is **cheap to verify or cheap to re-run.** A wrong grunt lane costs
  seconds, not a review cycle.
- The volume is what makes it annoying, not the difficulty.

Canonical grunt jobs:

- Run one simple check across 50 files ("does each of these have a frontmatter
  `title`?", "which of these import a deprecated symbol?").
- Mechanical transform ("rename this field in every fixture", "convert these
  30 notes' date format").
- Cheap classification / tagging ("label each of these 200 log lines as
  auth / network / disk / other").
- Bulk shallow scan and extract ("list every TODO with its file:line and the
  surrounding function name").
- First-pass triage that a better tier will then review.

**Do NOT route here:** anything contract-sensitive, security-relevant,
architecture-shaped, ambiguous, or where a subtly-wrong answer would be
expensive to catch. That's Codex `sol` at `high` or the main loop. Flash's failure mode is
confident plausible-looking wrongness on work that needed judgment — which is
exactly why it must only get work where wrongness is obvious and cheap.

## CRITICAL: This is an ACTION skill, not commentary

When invoked you MUST:

1. **Actually invoke `agy -p` via the Bash tool.** Never write instructions for
   the user to run themselves.
2. **Default to background execution** (`run_in_background: true`) for anything
   likely to take >10s. The harness notifies on completion.
3. **For multiple independent jobs, fire them ALL in parallel — but say so
   first.** Before firing more than one lane, state in ONE sentence how many
   lanes and which model each gets. Disclosure, not an approval gate.
4. **Summarize from logs** after each lane completes — don't dump raw stdout.

## USAGE RULES (read this before spawning anything)

This stack uses this tier **generously** by design. Quota is
still an account-level resource, so the rules below keep a fleet *visible* and
stop it on a real quota error — they are not a reason to under-use Gemini.
(Background: on the Codex side a silent 4-lane fleet on the most expensive model
once drained a quota in minutes; disclosure exists so that never happens
unnoticed.)

Binding rules:

- **N > 1 lanes requires a one-sentence disclosure first** ("firing 5 lanes, all
  `gemini-3.8-flash-low`"). Always. Even for cheap lanes.
- **Default model is `gemini-3.8-flash-low`; step up to `flash-medium` freely**
  when items need light reasoning or `low` output looks shaky. `flash-high` is
  fine when a lane clearly benefits.
- **Never `gemini-3.1-pro-*` from this skill.** If a job needs Pro-class
  reasoning it is not a grunt job — it belongs to Codex or the main loop.
  Routing it here spends Antigravity quota on work another tier does better.
- **Never `claude-*` or `gpt-oss-*` through `agy`.** They appear in
  `agy models` but routing Claude work through a third-party CLI is pointless
  indirection — use the native harness.
- **Verification calls are one call, cheapest model, shortest prompt.** Never
  "test the fleet" with a parallel run.
- **Batch where it's natural, fan out when it helps.** A 50-item list in one lane
  pays the ~13K per-call overhead once, so batching is still the efficient
  shape for tiny items. But split into several parallel lanes freely whenever
  it improves speed, keeps prompts focused, or isolates failures.
- **No personal data to Gemini.** Lane content goes to Google. Don't send notes
  holding personal identity/contact info (e.g. a private notes folder);
  pick another tier for those.

## Prerequisites

- `agy` installed and **interactively logged in** with a personal Google
  account. Verify cheaply:
  ```bash
  command -v agy && agy --version
  ```
  Verified locally 2026-09-16: `agy` at `~/.local/bin/agy`, version `1.2.4`.
- Auth check with **zero generation cost**: `agy models` hits the account and
  prints the entitled model list. If login is broken this fails — it is the
  correct cheap liveness probe, not a `-p` call.

## Models (verified live 2026-09-16 via `agy models`)

Reasoning effort is **baked into the model id** as a `-low` / `-medium` /
`-high` suffix. There is also a separate `--effort low|medium|high` flag.
**Verified live 2026-09-16, zero cost (exit 1, no tokens spent):** passing
both is a hard conflict, not an override —
`agy -p ... --model gemini-3.8-flash-low --effort high` fails with
`error: invalid model selection (--model "gemini-3.8-flash-low" --effort
"high"): --model gemini-3.8-flash-low conflicts with --effort=high`. **Encode
effort in the model id only. Never pass `--effort` alongside a suffixed model
id — it's not a fallback, it's a guaranteed error.**

| Model id | Use |
|---|---|
| `gemini-3.8-flash-low` | **Default for this skill.** Bulk grunt, classification, mechanical scans. |
| `gemini-3.8-flash-medium` | Grunt work with light per-item reasoning. Use freely when `low` looks shaky. |
| `gemini-3.8-flash-high` | Upper bound for this skill. Fine when a lane clearly benefits. |
| `gemini-3.7-flash-*`, `gemini-3.6-flash-*` | Older Flash generations; use only if a caller pins them or 3.8 misbehaves. |
| `gemini-3.1-pro-high` / `-low` | **Out of scope — do not route here.** See quota rules. |
| `claude-sonnet-4-6`, `claude-opus-4-6-thinking`, `gpt-oss-120b-medium` | Visible in the list; **not this skill's business.** |

## Base command

```bash
agy -p "<PROMPT>" \
  --model gemini-3.8-flash-low \
  --output-format text
```

Verified live 2026-09-16: `agy -p "reply with exactly: ok" --model
gemini-3.8-flash-low --output-format text` returned `ok` on the logged-in
session with **no API key set**. Headless print mode on account auth works.

### Flags that matter

| Flag | Meaning | Note |
|---|---|---|
| `-p` / `--print` / `--prompt` | run one prompt non-interactively and exit | the `codex exec` equivalent; **always use this in a fleet** |
| `--model <id>` | model for this session | always pass explicitly; never rely on the account default |
| `--output-format text\|json\|stream-json` | print-mode output shape | `text` and `json` both verified live 2026-09-16. **Prefer `json` for fleet lanes** — gives structured fields instead of scraping stdout. Envelope: `{conversation_id, status, response, error?, duration_seconds, num_turns, structured_output?, json_schema?, usage:{input_tokens, output_tokens, thinking_tokens, cache_read_tokens, total_tokens}}`. |
| `--json-schema <str\|file>` | enforce structured output | Verified live 2026-09-16 — works, populates `structured_output` in the JSON envelope. **Gotcha found live:** a transient `status:"ERROR"` (e.g. `UNAVAILABLE (503): No capacity`) can still carry a valid, schema-conforming `structured_output` in the same response — don't gate lane success purely on `status=="SUCCESS"`; check for `structured_output` presence first, fall back to a retry only if it's absent. Ideal for classification/grunt lanes — forces a parseable verdict. |
| `--add-dir <dir>` | add a directory to the workspace (repeatable) | how a lane sees files outside CWD |
| `--dangerously-skip-permissions` | auto-approve ALL tool calls incl. file writes and shell | the `--full-auto` / `danger-full-access` analogue — **write lanes only, after explicit user OK once per session** |
| `--sandbox` | run with terminal restrictions enabled | prefer this for read/scan lanes |
| `--mode plan\|accept-edits` | agent execution mode | `accept-edits` is the milder write posture |
| `--print-timeout` | default `5m0s` | raise for long bulk lanes: `--print-timeout 20m` |
| `--disable-slash-commands` | no slash/skill expansion in print mode | **use it in every fleet lane** — stops user data in the prompt from being parsed as a command |
| `-c` / `--continue`, `--conversation <id>` | resume | see Resume below |
| `--log-file <path>` | override CLI log path | useful for debugging a wedged lane |

### TRAP: `-i` is NOT an image flag here

In `codex-fleet`, `-i` attaches images and greedily eats the prompt unless you
pass `--`. In `agy`, **`-i` is `--prompt-interactive`** — it runs the prompt and
then *stays in an interactive session*. Passing `-i` in a background fleet lane
gives you a hung process holding a TTY, not an error. **Never use `-i` in this
skill. Always `-p`.**

### Read lane vs write lane

| Lane type | Flags |
|---|---|
| Read / scan / classify (default) | `--sandbox --disable-slash-commands` |
| Apply mechanical edits | `--dangerously-skip-permissions --disable-slash-commands` (confirm with user once per session before first use) |

Give a lane only what it needs. A classification lane must never get
`--dangerously-skip-permissions`.

### Exit codes

Clean run exits `0`. Failure to produce a response exits non-zero with the
reason on stderr. In lanes, capture both: `> /tmp/lane-A.log 2>&1`, then check
the log's tail before trusting the lane.

## Background-first invocation

```
Bash tool call:
  command: agy -p "List every file under knowledge/concepts/ whose frontmatter
           is missing a `title` key. Output one path per line, nothing else."
           --model gemini-3.8-flash-low --sandbox --disable-slash-commands
           --print-timeout 20m > /tmp/gem-frontmatter.log 2>&1
  run_in_background: true
```

Then keep working. On the completion notification, read the log and summarize.

## Resume

```bash
agy -p "follow-up prompt" -c --model gemini-3.8-flash-low
```

`-c` / `--continue` picks up the most recent conversation; `--conversation <id>`
targets a specific one. In a parallel fleet, **`-c` is ambiguous** — several
lanes finishing near each other make "most recent" a race. For fleet follow-ups
use `--conversation <id>` or just fire a fresh self-contained prompt. Fresh is
usually right for grunt work; these lanes are disposable by design.

---

## Fleets (parallel multi-lane grunt orchestration)

A **fleet** is N `agy -p` delegates working at once. Each lane is a plain
background process. Pure local orchestration — you spawn the lanes, hand each a
self-contained brief, watch the logs, integrate.

**Fleet or single lane?** Grunt work is usually *list-shaped*; tiny items batch
well into one lane (one ~13K overhead instead of fifty). Fan out whenever:

- per-item context is large, or
- lanes must WRITE to disjoint files concurrently, or
- one slow item would block the rest, or
- splitting the list into a few chunks simply gets the answer faster.

Either way: **structured output** so rows can be counted.

### Operator defaults for grunt lanes

| Setting | Default | Why |
|---|---|---|
| Model | `gemini-3.8-flash-low`, or whatever the caller stamped | Routing is the calling agent's decision (see `fleet-orchestration`); stepping up to `flash-medium` is cheap and fine. |
| Sandbox | `--sandbox` for read lanes, `--dangerously-skip-permissions` for write lanes | Only grant what the lane needs. |
| Slash commands | `--disable-slash-commands` always | Lane briefs contain user data; don't let it expand. |
| Timeout | `--print-timeout 20m` for bulk lanes | The 5m default silently truncates long scans. |
| Working dir | `-C` does not exist in `agy` — use a `cd` in the spawned shell plus `--add-dir <dir>` | This is the notable divergence from `codex-fleet`. |
| Lane count | as many as the job benefits from; disclose first | Generous by design; stop only on a real quota/rate-limit error. |

### Spawn recipe (one lane)

```bash
caffeinate -i sh -c 'cd <LANE_DIR> && agy -p "<SELF-CONTAINED LANE BRIEF>" \
  --model gemini-3.8-flash-low \
  --add-dir <LANE_DIR> \
  --sandbox \
  --disable-slash-commands \
  --print-timeout 20m' > /tmp/gem-lane-A.log 2>&1
```

Fire with `run_in_background: true`.

> **`caffeinate -i` (macOS).** A lid-close or idle sleep silently kills a
> mid-flight lane — you get a truncated log and zero work. Wrap every spawn.
> A respawn with the same brief is safe when `git status` shows no partial work.

### The lane brief contract

The brief is the lane's **entire** contract — a delegate cannot see your
conversation. For a Flash grunt lane, be *more* explicit than you would be for
Codex: it follows instructions well but improvises badly and will happily
invent structure you didn't ask for.

Every brief states:

1. **Goal** — one sentence, one job. No compound tasks.
2. **The exact items** — the file list, the line list, the input, inline.
   Don't say "all the notes"; enumerate or give a precise glob.
3. **OWNS** — the exact files this lane may write (write lanes only).
4. **DO-NOT-TOUCH** — files owned by a sibling, naming which sibling.
   Lanes report fouls ("sibling edits present, left intact"); they never "fix"
   them.
5. **Output format, verbatim** — a grunt lane's value is machine-readable
   output. Specify it exactly and say "nothing else": one `path\tverdict` per
   line, or strict JSON (pair with `--json-schema` once verified).
6. **Acceptance check** — what the lane must confirm before reporting done.
7. **Report line** — the lane's last line must be exactly
   `DONE: <one-line summary>` or `FAILED: <reason>`.

Anti-hallucination clause worth pasting into every brief:

```
If you cannot determine the answer for an item from the actual file contents,
output UNKNOWN for that item. Do NOT guess, do NOT infer from the filename, and
do NOT omit the item. Every input item must appear exactly once in the output.
```

That last sentence is the cheap correctness handle: count the output lines
against the input list, and a dropped or duplicated item is an instant tell
that the lane went off the rails.

### Fleet patterns

- **Stagger the spawns 2–5s apart.** Firing every lane's first model call at
  once is a thundering herd; lanes wedge on dead connections at startup and sit
  silent. The stagger costs a minute; a zombie costs half an hour.
- **Scale lanes to the job** and state the count out loud. 2 concurrent lanes
  are verified clean; go higher as needed and watch for `UNAVAILABLE`/`503`/
  rate-limit errors — back off only when one actually appears.
- **Read lanes stay sandboxed.** A scan lane gets `--sandbox`, never
  `--dangerously-skip-permissions`. If it turns out edits are needed, escalate
  to a separate write lane — don't tell a read lane to patch.
- **Liveness check from the surface side:** a lane whose log hasn't grown in
  many minutes is dead regardless of the process table. Respawn with the same
  brief.
- **Completions are claims, not evidence.** `DONE:` means the lane *thinks* it
  finished. Run the acceptance check yourself: line count vs input count, spot
  check 2–3 items against the real files, then integrate. For a grunt tier this
  is non-negotiable — cheap output that nobody verified is worse than no output.

### Concurrent WRITE lanes — worktree isolation

Same trick as `codex-fleet`. When multiple lanes must WRITE to the same repo at
once they clobber each other in a shared checkout.

**A. Worktree-per-lane + one commit per lane + orchestrator cherry-picks**
(use whenever lanes touch the same file or collisions are plausible):

```bash
git worktree add --detach ../gemfleet-lane-A <BASE_SHA>
# Spawn the lane with: cd ../gemfleet-lane-A && agy -p "..." --add-dir "$PWD" ...
# The brief tells the lane to make ONE commit when done.
# Orchestrator cherry-picks each lane's commit in completion order, gating at
# each pick. Prove the gate is GREEN on a baseline worktree before any spawn.
```

**B. Shared tree with explicit OWNS / DO-NOT-TOUCH lists** (fine when lanes
touch clearly disjoint files):

- Every brief carries OWNS + DO-NOT-TOUCH naming the sibling owner.
- Shared files (barrels, indexes, front-page notes) are **append-only** and
  allocated to exactly one lane at integration time.
- **Lanes never run the repo-root gate and never run git** (worktree mode
  excepted, where they make exactly one commit). The orchestrator runs the
  single full gate after all lanes settle, repairs cross-lane breakage, then
  commits per coherent unit.

### What makes a grunt fleet run good

- **Right-size the lanes.** Batch tiny items (overhead is per call), fan out
  when it buys speed or isolation. Don't under-use the tier.
- **Structured output or it didn't happen.** Free-prose output from a Flash lane
  is unverifiable at volume. Demand a strict line/JSON format and count the rows.
- **Verify a sample, always.** Spot-check 2–3 items per lane against the real
  files before integrating. The tier is cheap because you keep it honest.
- **Honest failure beats a cheerful lie.** `FAILED: couldn't read 12 of the 50
  paths` is a useful lane. A lane that quietly returns 38 rows for a 50-row
  input is the failure mode this whole file is built to catch.
- **Expect other work on the machine.** Only act on your own fleet's lanes and
  logs; never reap processes you didn't spawn.

## Error handling

- If `agy --version` or a lane exits non-zero, **stop and report.** Do not retry
  blindly — a retry loop on a quota error burns what's left of the quota.
- If stderr mentions quota / rate limits, **halt the whole fleet**, report to the
  user, and do not respawn. This is the one failure that must never be handled
  by "try again."
- `--dangerously-skip-permissions` requires explicit user OK before its first use
  in a session; after that it's fine within the same task scope.
- If login has lapsed, `agy models` fails — tell the user to re-run the
  interactive `agy` login. Do **not** work around it by setting an API key;
  this skill is built for account login, not API keys.

## Verified 2026-09-16 (was open, now resolved live)

- **`--effort` vs the model-id suffix:** hard conflict, confirmed at zero cost
  (see Models section above). Never combine them.
- **`--output-format json` envelope + `--json-schema`:** both confirmed live,
  including the `status:"ERROR"`-with-valid-`structured_output` gotcha (see
  Flags table above).
- **Fixed per-call overhead is large relative to a grunt prompt.** A 5-word
  prompt (`"reply with exactly: ok"`) cost **~13,200 input tokens** — almost
  entirely `agy`'s own system/tool-context overhead, not the prompt. This
  overhead is roughly constant regardless of prompt size, which is why
  batching tiny items is efficient: one lane holding 50 items pays the ~13K
  overhead once; 50 one-item lanes pay it 50 times over. Chunk accordingly —
  a few parallel chunks, not one lane per trivial item.
- **Concurrency:** 2 simultaneous `agy -p` background processes (both
  `gemini-3.8-flash-low`) completed cleanly with no contention, no throttling,
  no dropped output. **Still open:** the ceiling above 2 lanes, and where the
  account-level quota wall actually sits. Scale up as the job needs and watch
  for `UNAVAILABLE`/`503`/rate-limit errors.
- **Sample real lane (2026-09-16):** 1 lane, `flash-low`, `--sandbox`,
  6-file frontmatter extraction → 6/6 correct, 104 s, ~86K tokens (~40K cached).
