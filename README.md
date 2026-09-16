# agent-fleet-skills

Agent skills for running a **multi-model coding fleet on ~$20/month plans**.
Claude Code is the orchestrator. Codex CLI and Gemini (via the Antigravity CLI)
are delegate workers. Every piece of work gets routed to the cheapest
subscription that can do it right.

> **Built for $20 plans, tuned for price/performance.** These skills assume you
> do *not* have a top-tier subscription anywhere. Instead of one expensive plan
> that does everything, you combine three cheap ones and treat each plan's
> quota as a separate budget. The routing rules exist to protect the scarcest
> budget and push volume onto the cheapest one that's still correct.

## The idea

| Pool | Plan | Role | Spend it on |
|---|---|---|---|
| **Claude** | ~$20 Claude plan | main loop + Claude sub-agents | judgment, specs, architecture, final review. Scarcest pool |
| **ChatGPT** | ~$20 ChatGPT Plus | Codex lanes | precise execution of a well-written spec |
| **Google** | personal Google account | Gemini Flash lanes | uniform, high-volume grunt work. Used generously |

What that means in practice:

- **Claude sub-agents share the main loop's quota.** They always get an
  explicit, cheapest-capable model. They're never a free way to parallelize.
- **Codex runs everything on `gpt-5.6-sol`** and expresses difficulty through
  reasoning effort (`medium` → `high` → one `xhigh` lane). `gpt-6-astra` is
  powerful but drains a Plus plan fast, so it's **never auto-routed**. It only
  runs when you ask for it.
- **Gemini Flash is the bulk pool.** "Check these 50 files", "label these 200
  lines", "same mechanical edit everywhere" go here, with structured output so
  the results can be counted and spot-checked.
- **The most critical piece stays in the main loop.** Higher effort on a cheaper
  model buys more thinking, not a higher ceiling.

## Skills

| Skill | What it does |
|---|---|
| [`fleet-orchestration`](skills/fleet-orchestration/SKILL.md) | The routing policy: which pool, which model, which effort, and why. Load it before any delegation. |
| [`codex-fleet`](skills/codex-fleet/SKILL.md) | Runs `codex exec`: single lanes, parallel fleets with worktree isolation, and image generation through Codex's built-in `gpt-image-2` tool. |
| [`gemini-fleet`](skills/gemini-fleet/SKILL.md) | Runs `agy -p` (Antigravity CLI) for grunt lanes: batched lists, structured output, sandboxed read lanes. |
| [`gptpro`](skills/gptpro/SKILL.md) | Exports a repo into clean, secret-scanned, subsystem-split zip bundles for a non-agentic chat model. |
| [`gptpro-handoff`](skills/gptpro-handoff/SKILL.md) | Workflow for handing one hard problem to a slow, deep chat-UI model, then verifying every finding against the live code before implementing. |

Every executor skill follows the same rules:

- Commands actually run in the background.
- Before starting more than one lane, the orchestrator says in one sentence how many lanes and which model/effort each gets.
- A lane reporting "done" is only a claim. The orchestrator checks its work before integrating.

## Requirements

- [Claude Code](https://claude.com/claude-code) (orchestrator)
- [Codex CLI](https://github.com/openai/codex) logged in with a ChatGPT account
  (no API key needed)
- Antigravity CLI (`agy`) logged in with a Google account (no API key needed)
- `zip` and `rsync` for `gptpro`
- macOS or Linux. Fleet lanes use `caffeinate -i` on macOS to survive sleep.

Model IDs and CLI flags were verified live in **September 2026**. Providers
rename and retire models often, so check `agy models` and your Codex model list
if something stops resolving.

A note on `gptpro-handoff`: the deepest "Pro" reasoning models in the ChatGPT
web UI may require a higher plan than Plus. The workflow still works with the
strongest thinking model your plan offers.

## Install

```bash
git clone https://github.com/enesadakli/agent-fleet-skills.git
cd agent-fleet-skills
mkdir -p ~/.claude/skills
for s in skills/*; do ln -sfn "$PWD/$s" ~/.claude/skills/"$(basename "$s")"; done
```

Symlinks keep the installed skills in sync with `git pull`. Other agents that
read `SKILL.md` folders can use the same directories.

## Adapting it

Plans, quotas and model lineups differ. The policy lives in one place,
`fleet-orchestration`, so you can change routing there without touching the
executors. With a bigger ChatGPT plan, for example, you might let `astra` take
hard lanes again. Personal preferences, like how generously to use a pool or
which data may leave your machine, belong in your own project's agent
instructions, not in these files.

## Türkçe özet

Bu skill'ler **20 dolarlık aboneliklerle** çok modelli bir ajan filosu kurmak
için yazıldı. Tek bir pahalı plan yerine üç ucuz plan (Claude, ChatGPT Plus,
Google hesabı) birlikte kullanılıyor ve her planın kotası ayrı bir bütçe olarak
yönetiliyor. Karar, spec ve son kontrol Claude ana oturumunda kalıyor.
Spec'i yazılmış işler Codex `sol` ile yapılıyor, zorluğa göre effort artıyor.
`astra` yalnızca açıkça istenirse kullanılıyor. Toplu ve tekdüze işler Gemini
Flash'a cömertçe veriliyor. Amaç fiyat/performans: her iş, onu doğru yapabilen
en ucuz kotaya gidiyor.

## License

MIT. See [LICENSE](LICENSE).
