# Code Dialectic

A [code-conductor](https://github.com/UnmanagedCode/code-conductor) plugin that runs a structured **thesis → antithesis → synthesis** dialectic between two spawned workers, to stress-test an idea, decision, or design against its strongest objections.

## What it does

Given a topic, question, or design problem, the plugin's conductor-facing skill:

1. Frames a precise, falsifiable **proposition**.
2. Spawns two `dialectician`-role workers — **Thesis** (argues for) and **Antithesis** (argues against).
3. Relays their turns back and forth for several rounds, each required to steelman the opponent, concede valid points, and end with a stance summary.
4. Runs a final **synthesis** round: both workers draft their refined position, and the conductor composes the final answer — preferring what survived scrutiny, and honestly surfacing any residual disagreement rather than splitting the difference.
5. Reports the proposition, a round-by-round summary, and the final synthesis, ending with the sentinel `DIALECTIC_COMPLETE`.

### How to invoke

- Slash command: `/code-dialectic:dialecticate <topic or question>`
- Auto-trigger: just ask for a dialectic on a topic, to debate both sides of a decision, to stress-test an idea through adversarial argument, or to reach a position that survives the strongest objections — the conductor picks up the skill on its own.

### The `dialectician` role

A named model-binding (no persona) resolvable as `code-dialectic/dialectician` anywhere roles are used (e.g. `spawn_instance`'s `model` arg). It's bound to the `powerful` capability tier by default. Stance (Thesis/Antithesis) comes entirely from the conductor's prompts each round, not from the role itself.

- **Enable**: Settings → Plugins.
- **Model binding**: the `dialectician` role is **visible** in Settings → Models → Roles with a "via code-dialectic" badge, but it is **read-only** there — not user-rebindable or deletable. To change its model binding, edit `roles[].binding` in `conductor.plugin.json` and re-enable the plugin.

## Technical description

Contributions-only plugin — no backend, no frontend, no MCP tools. It contributes a `roles` entry and a `claudePlugin` root via `conductor.plugin.json`; nothing is ever started as a process.

### File layout

```
conductor.plugin.json          # cc plugin manifest (roles + claudePlugin)
.claude-plugin/plugin.json     # Claude Code plugin descriptor
skills/dialecticate/SKILL.md   # the conductor-facing dialectic playbook
```

### Manifest

`conductor.plugin.json` declares:
- `roles: [{ slug: "dialectician", name: "Dialectician", binding: { kind: "tier", tier: "powerful" } }]` — joins the merged role list namespaced `code-dialectic/dialectician`.
- `claudePlugin: "."` — the plugin project root doubles as the Claude Code plugin root, so `skills/dialecticate/SKILL.md` resolves as a normal Claude Code skill.

Because `claudePlugin` is set, every enabled + `ok` instance of this plugin gets one `--plugin-dir <resolved-root>` flag added at **every** claude launch (interactive sessions and MCP-spawned workers alike), which is how `/code-dialectic:dialecticate` becomes available as a slash command and how the skill's auto-invoke description reaches Claude's routing.

### Orchestration mechanics

The skill never authors arguments itself — it drives two `dialectician` workers via **dispatch-and-wake**: `send_prompt({ sessionId, subscribe: true })` followed by ending the conductor's turn, resubscribing on each wake for as many rounds as the dialectic runs (default 3, adjustable). Each round relays the opponent's latest turn verbatim into the other worker's next prompt, alongside a restated rules block, so nothing drifts across turns.

### Known limitations

- **Cost scales with rounds × 2 workers** — each round is a full turn from both Thesis and Antithesis, so a longer dialectic (or a "be exhaustive" request) multiplies spawn/prompt cost accordingly.
- **The role is a model-binding only** — `dialectician` carries no persona or system prompt; all stance-taking (Thesis vs. Antithesis) and rule enforcement comes from the conductor's prompts, not the role itself.

## License

This project is licensed under the GNU Affero General Public License v3.0 or later (AGPL-3.0-or-later).
© 2026 UnmanagedCode. See [LICENSE](LICENSE) for details.
