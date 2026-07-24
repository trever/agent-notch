# Agent Notch vs Vibe Island — feature audit

Audit date: 2026-07-23. Vibe Island 1.0.42 (`app.vibeisland.macos`), inspected from
`VibeIsland.dmg`: bundle layout, helper binaries, and its 800-key `Localizable.strings`,
which is the most complete inventory of its feature surface.

## The one difference that determines everything else

|  | Vibe Island | Agent Notch |
| --- | --- | --- |
| Liveness | **CLI hooks** — ships `vibe-island-hook-{darwin,linux,freebsd}-{amd64,arm64}` and installs them into each agent's config | `ps` + `lsof` process discovery |
| State detail | Hook events: turn start/stop, pre/post tool use, permission requests, questions | Transcript JSONL tail + process liveness |
| Setup cost | Writes to agent configs; has a `hookWatcher` to re-install when they get removed, and a "restart your sessions" banner | Zero config, nothing installed |
| Also uses | `vibe-island-bridge` helper, a Claude **statusline bridge** for usage data, AppleEvents for terminal control | — |

Nearly every gap below traces back to this. Hooks give Vibe Island *events*; we infer
*state* by polling. Anything needing "the exact instant X happened" or "respond to the
agent" needs hooks. Anything that's a fact about the world (what tool is running, which
terminal owns a session) we can get without them.

**The strategic question for the roadmap:** our README calls the no-hooks design
deliberate. Several of the highest-value items below are hooks-only. The likely answer is
an *optional* hooks mode — zero-config by default, opt in for precision — rather than
abandoning either position.

## Feature-by-feature

Legend — **Have**: shipped in our fork. **Gap**: Vibe Island has it, we don't.
Effort: S (hours) · M (a day) · L (multi-day) · XL (a project).

### Notch & panel presentation

| Feature | Vibe Island | Us | Notes / effort |
| --- | --- | --- | --- |
| Black island fused to hardware notch | ✅ | ✅ | Shipped today |
| Collapsed one-line "what's happening" | ✅ | ✅ | Ours derives it from transcript `tool_use` |
| Expand on hover + hover duration setting | ✅ | ✅ (fixed 350 ms) | Setting needs a prefs UI — S |
| Auto-collapse on mouse leave | ✅ toggle | ✅ (always, if hover-opened) | S to make it a setting |
| Session badge / count | ✅ | ✅ | |
| Subagents in panel | ✅ toggle | ✅ | |
| Model tag | ✅ toggle | ✅ | |
| Project name / worktree tags | ✅ toggles | Project only | Worktree tag — S |
| Per-session elapsed time (`<1m`, `27m`) | ✅ | ❌ | We have the data — **S, high value** |
| Terminal/app tag per row (`iTerm`, `Ghostty`) | ✅ | ❌ | We know the tty/app — **S** |
| Per-row activity line under the prompt | ✅ | ❌ | We already compute it for the island — **S** |
| Task/todo list panel (`8 done, 1 in progress`) | ✅ | ❌ | Parseable from `TodoWrite` in transcript — **M** |
| Layout modes (Clean / Detailed) | ✅ | ❌ | M |
| Sizing: notch w/h, panel w/h, font size, card height | ✅ sliders | ❌ | Needs prefs UI — M |
| Auto-hide when idle | ✅ | ✅ (draws nothing) | |
| Hide in fullscreen | ✅ | ❌ (we go full-width) | S |
| Multi-display / "Main Display" picker | ✅ | ❌ (hardcodes origin screen) | M |

### Agent coverage

| Feature | Vibe Island | Us | Notes |
| --- | --- | --- | --- |
| Claude Code | ✅ | ✅ | |
| Codex | ✅ | ✅ | |
| Claude desktop app sessions | ✅ | ✅ | Shipped today |
| Cursor | ✅ (incl. sandbox approvals) | Partial — we see its Codex, no special handling | M |
| Gemini, Kimi, Z.ai, Trae, Kiro | ✅ (icons + hook configs) | ❌ | Each is its own transcript format — M each |
| Custom CLI branches / forks / paths | ✅ | ❌ | M |

### Interaction

| Feature | Vibe Island | Us | Notes |
| --- | --- | --- | --- |
| Click island → panel | ✅ | ✅ | |
| **Click session → jump to its terminal window/tab** | ✅ | ❌ | Uses AppleEvents (`NSAppleEventsUsageDescription`). **No hooks needed — M, very high value** |
| Custom jump rules | ✅ | ❌ | M |
| Global shortcuts / session switcher | ✅ | ❌ | M |
| Panel keyboard nav | ✅ | ❌ | M |
| **Approve/deny permission prompts from the notch** | ✅ | ❌ | Hooks + `--permission-prompt-tool`. **XL, hooks-only** |
| Answer agent questions (multi-select wizard) | ✅ | ❌ | Hooks-only — L |
| Exit-plan-mode UI (Auto/Bypass/Manual) | ✅ | ❌ | Hooks-only — L |

### Notifications & attention

| Feature | Vibe Island | Us | Notes |
| --- | --- | --- | --- |
| Completion notifications | ✅ | ✅ green blob only | Real notifications — S |
| Sound themes, 8-bit pack, custom CESP packs | ✅ | ❌ | S for one sound, L for the pack system |
| Auto-expand panel on completion | ✅ | ❌ | S |
| Subagent notification timing policy | ✅ 3 modes | ❌ | S once notifications exist |
| Quiet scenes: Focus mode, screen locked, screen sharing | ✅ | ❌ | **S–M, genuinely thoughtful** |
| Idle-session cleanup interval | ✅ 30 m–never | ✅ fixed 6 h | S to make configurable |
| Session admission rules (block by dir / first prompt / launcher app) | ✅ | ❌ | M |

### Usage & telemetry

| Feature | Vibe Island | Us | Notes |
| --- | --- | --- | --- |
| Usage-limit header (`5h 11%`, `7d 2%`) | ✅ via **Claude statusline bridge** | ❌ | Chains `~/.claude/settings.json` statusLine. **M, no hooks needed** |
| Usage threshold alerts | ✅ | ❌ | S after the above |
| Codex usage-limit messaging | ✅ | ❌ | S |

### Platform / distribution

| Feature | Vibe Island | Us | Notes |
| --- | --- | --- | --- |
| Settings window | ✅ (280 keys!) | ❌ (a pet file) | **The single biggest structural gap — L** |
| Launch at login | ✅ menu toggle | ❌ (manual) | `SMAppService` — S |
| Sparkle auto-update | ✅ | ❌ | We have a skill for this — M |
| Localization (10 languages) | ✅ | ❌ | Skip |
| Sentry crash reporting, diagnostics export | ✅ | ❌ | Skip for personal use |
| **SSH remote sessions** (85 keys, port fan-out) | ✅ | ❌ | Monitor agents on remote boxes — XL |
| AI session auto-naming (Labs) | ✅ | ❌ | Needs an LLM call — M |
| Licensing / paywall ($19.99 license, $14.99 unlock) | ✅ | n/a | Not applicable |

## Suggested plan

**Tier 1 — cheap, no hooks, high payoff (do first)**
1. Per-row activity line + elapsed time + terminal tag in the panel — we already have all
   three facts; it's presentation only.
2. Click a session → jump to its terminal window/tab. Probably the single biggest
   day-to-day usability win Vibe Island has over us, and it needs no hooks.
3. Launch at login via `SMAppService`.
4. Completion notification + one sound.
5. Hide in fullscreen.

**Tier 2 — structural**
6. A real settings window. Everything above accumulates knobs with nowhere to live, and
   280 of Vibe Island's keys are settings. Until this exists, every new option is
   hardcoded.
7. Claude usage bridge for the limits header — self-contained and no hooks.
8. Quiet scenes (Focus/locked/screen-sharing).
9. Task-list panel from `TodoWrite`.

**Tier 3 — the hooks decision**
10. Optional hooks mode: opt-in, fixes the ~30 s afterglow the README already flags as the
    known limitation, and unlocks 11.
11. Permission approvals from the notch. The marquee feature, and Vibe Island charges for
    it. XL, and it means owning the `--permission-prompt-tool` path.

**Probably skip:** localization, Sentry, licensing, SSH remote (unless you actually run
remote agents), AI session naming.

## Legal note

Vibe Island is a paid, closed-source product. This audit is a black-box read of its
*feature list* from user-visible strings, to decide what we want — not a basis for copying
its implementation. Our own pet spritesheets are already OpenAI's Codex assets (fine
personally, worth revisiting if this fork ever goes public).
