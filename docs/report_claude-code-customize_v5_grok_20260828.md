# Customizing Claude Code CLI and Codex CLI for Long-Running AI-SWE / AI-SDLC Sessions

**Document:** `report_claude-code-customize_v5_grok_20260828.md`  
**As-of date:** 2026-08-28  
**Authoring agent:** Grok 4.6 (xAI)  
**Scope:** Concrete, production-oriented customization of Anthropic **Claude Code CLI** and OpenAI **Codex CLI** for hour-to-multi-day agentic coding, AI-SWE, and AI-SDLC work.

This is an implementation report, not a marketing overview. Every recommendation includes *where* to put the file, *what* to paste, *why* it matters for long sessions, and a source URL.

---

## 0. Executive rules of thumb

1. **Put “always true” project facts in disk-backed files** (`CLAUDE.md` / `AGENTS.md`, auto-memory, plans). Conversation text is lossy after compact/resume.
2. **Use hooks for anything that must happen the same way every time.** Skills and markdown are suggestions. Hooks are deterministic.
3. **Keep the always-loaded surface small.** Skill *descriptions* load every session. Skill *bodies* should load on demand. CLAUDE.md should stay under ~200 lines (official guidance) and never become a novel.
4. **Compact at task boundaries, not at crisis.** Auto-compact fires when the model is already degraded. Manual `/compact focus on …` at 50–70% is higher quality than waiting for the red bar.
5. **New task → new session.** A 1M window does not make “everything in one chat” a good idea. Context rot still happens.
6. **Delegate read-heavy work to subagents.** Investigation noise stays in the child context.
7. **Deny secrets in permissions; don’t just write “never read `.env`” in markdown.**
8. **Commit shared project config; gitignore personal overrides.**
9. **Token-compress shell output before it enters the window** (RTK / ContextZip). Most session bloat is command output, not source.
10. **Route through a gateway only if you need fallbacks, spend caps, or non-Anthropic models.** Gateways add failure modes; set the context-window correction or compaction will fire at the wrong time.

---

## 1. Official documentation map (bookmark these)

### Claude Code

| Topic | URL |
|---|---|
| Settings (precedence + files) | https://docs.claude.com/en/docs/claude-code/settings.md |
| Settings reference | https://code.claude.com/docs/en/settings-reference |
| Example settings files | https://code.claude.com/docs/en/settings-example |
| Hooks reference | https://code.claude.com/docs/en/hooks |
| Hooks guide (first hook) | https://code.claude.com/docs/en/hooks-guide |
| Skills | https://code.claude.com/docs/en/skills |
| Context window + compaction | https://code.claude.com/docs/en/context-window |
| Sessions / resume / retention | https://code.claude.com/docs/en/sessions |
| Memory (`MEMORY.md`) | https://code.claude.com/docs/en/memory |
| Steering: CLAUDE.md vs skills vs hooks vs subagents | https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more |
| Session management + 1M context | https://claude.com/blog/using-claude-code-session-management-and-1m-context |
| Power-user tips | https://support.claude.com/en/articles/14554000-claude-code-power-user-tips |
| LLM gateway | https://docs.claude.com/en/docs/claude-code/llm-gateway |
| Features overview / when to add what | https://code.claude.com/docs/en/features-overview |

### Codex CLI

| Topic | URL |
|---|---|
| Hooks | https://developers.openai.com/codex/hooks/ |
| Advanced config | https://developers.openai.com/codex/config-advanced |
| Slash commands (`/goal`, `/hooks`, `/clear`) | https://developers.openai.com/codex/cli/slash-commands |
| Agent loop / AGENTS.md load order | https://openai.com/index/unrolling-the-codex-agent-loop/ |
| Goal mode (hours–days persistence) | https://youtu.be/rgh0hMYPcd0 |

### High-signal community / templates

| Resource | URL |
|---|---|
| Trail of Bits hardened `settings.json` (`cleanupPeriodDays: 365`) | https://github.com/trailofbits/claude-code-config |
| Claude Code guide (rules/skills/hooks installer) | https://github.com/ytrofr/claude-code-guide |
| Settings + hooks practical guide | https://claudefolio.com/guides/claude-code-settings-hooks-permissions |
| Settings reference (third-party, dense) | https://claudefa.st/blog/guide/settings-reference |
| Context buffer / autocompact override | https://claudefa.st/blog/guide/mechanics/context-buffer-management |
| RTK (Rust Token Killer) | https://github.com/rtk-ai/rtk |
| ContextZip | https://github.com/jee599/contextzip |
| LiteLLM + Claude Code | https://docs.litellm.ai/docs/tutorials/claude_non_anthropic_models |
| LiteLLM auto-router | https://docs.litellm.ai/docs/tutorials/claude_code_autorouter |
| Codex session storage layout | https://allaboutcoding.ghinda.com/where-ai-coding-clis-store-session-logs/ |

---

## 2. File layout you should actually use

### Claude Code

```
~/.claude/settings.json              # personal defaults (all projects)
~/.claude/skills/<name>/SKILL.md     # personal skills
~/.claude/hooks/*.sh                 # personal hook scripts
~/.claude/projects/<slug>/*.jsonl    # transcripts (default 30-day cleanup)

<repo>/CLAUDE.md                     # always-on project briefing (keep short)
<repo>/.claude/settings.json         # team: permissions, hooks, plugins, env
<repo>/.claude/settings.local.json   # YOU only — gitignore this
<repo>/.claude/skills/<name>/SKILL.md
<repo>/.claude/hooks/*.sh
<repo>/.claude/rules/*.md            # path-scoped rules
<repo>/.mcp.json                     # project MCP servers (treat as code)
```

**Precedence (highest wins), official as of 2026-08-28:**

1. Managed settings (org / MDM / claude.ai console)  
2. CLI: `claude --settings '{...}'`  
3. Project local: `.claude/settings.local.json`  
4. Shared project: `.claude/settings.json`  
5. User: `~/.claude/settings.json`

**Special merge rules:**

- `deny` permission rules from *any* layer beat `allow` from any other layer.
- `hooks` **merge** across layers; they do not replace.
- Managed hooks cannot be removed by user/project files.
- Settings files are watched and reload live (`permissions`, `hooks`, `apiKeyHelper`). A `ConfigChange` hook fires on reload.

Install CLI: `curl -fsSL https://claude.ai/install.sh | sh`  
Docs: https://docs.claude.com/en/docs/claude-code/settings.md

### Codex CLI

```
~/.codex/config.toml                 # user defaults (TOML, not JSON)
~/.codex/<profile>.config.toml       # profiles: codex --profile deep
~/.codex/hooks.json                  # user hooks
~/.codex/AGENTS.md                   # global agent instructions
~/.codex/skills/**/SKILL.md
~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl   # full transcripts
~/.codex/history.jsonl               # prompt-only history (NOT the full rollout)
~/.codex/memories/                   # async session consolidation

<repo>/AGENTS.md                     # project briefing (walks root → cwd)
<repo>/.codex/config.toml
<repo>/.codex/hooks.json
```

Docs: https://developers.openai.com/codex/config-advanced

---

## 3. Step-by-step: Claude Code baseline for multi-day work

### Step 3.1 — Create the user settings file

Path: `~/.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "claude-sonnet-5",
  "effortLevel": "high",
  "cleanupPeriodDays": 365,
  "autoMemoryEnabled": true,
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git add *)",
      "Bash(npm run *)",
      "Bash(pnpm *)",
      "Bash(cargo test *)",
      "Bash(pytest *)",
      "Read",
      "Glob",
      "Grep"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(git commit *)",
      "Bash(curl *)",
      "Bash(docker *)",
      "WebFetch"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Bash(rm -rf *)",
      "Bash(sudo *)",
      "Bash(git push --force *)"
    ]
  },
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "50"
  },
  "statusLine": {
    "type": "command",
    "command": "jq -r '\"[\\(.model.display_name)] \\(.context_window.used_percentage // 0)% ctx\"'",
    "padding": 2
  }
}
```

**Why these values for long runs:**

| Key | Recommended | Why |
|---|---|---|
| `cleanupPeriodDays` | `365` (default **30**) | Multi-day / multi-week SDLC needs `/resume` and `/insights` over months, not 30 days. Official key: https://code.claude.com/docs/en/sessions |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | `50` for fragile long tasks; `70–80` if you want more runway | Default auto-compact is late (~83.5% of a 200K window after a ~33K buffer). Compacting earlier produces a better summary because the model is not yet rotten. Community mechanics: https://claudefa.st/blog/guide/mechanics/context-buffer-management |
| `effortLevel` | `high` or `xhigh` for architecture / multi-file refactors; drop to `medium` for grunt edits | Long sessions waste money if every turn is max-effort. Switch per task with `claude --settings '{"effortLevel":"xhigh"}'`. |
| `autoMemoryEnabled` | `true` | `MEMORY.md` + topic files survive compaction and are re-injected from disk. https://code.claude.com/docs/en/memory |
| Permission allow/deny | Real lists, not “skip all permissions” | Approval fatigue is the #1 reason people disable safety. Pre-allow *safe* commands; keep push/force/rm/sudo on `ask` or `deny`. |

**Transcript location (do not lose this):**

```
~/.claude/projects/<sanitized-cwd>/<session-id>.jsonl
```

Move the whole config+history tree with:

```bash
export CLAUDE_CONFIG_DIR=/srv/claude-config
```

Docs: https://code.claude.com/docs/en/sessions

**Alternative compaction controls (same family, different knobs):**

```text
/autocompact 500k          # fire auto-compact at an absolute token count
/compact focus on the auth refactor, drop test-debug noise
/context                   # live breakdown + optimization hints
/clear                     # new task, keep old session resumable
```

You can also put compaction policy *in CLAUDE.md* so every compact (manual or auto) is steered:

```markdown
## Context Compaction Rules
When compacting, always preserve:
- Current goal and definition of done
- Files modified this session and why
- Failing tests and exact reproduction commands
- Architectural decisions and rejected alternatives
- Open questions / blockers
Drop: raw tool dumps, successful test logs, exploratory dead-ends.
```

Official compaction survival table: https://code.claude.com/docs/en/context-window

### Step 3.2 — Team project settings (commit this)

Path: `<repo>/.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Bash(npx prettier --write *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Bash(rm -rf *)",
      "Bash(git push --force *)"
    ],
    "ask": ["Bash(git push *)"]
  },
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-rm.sh",
            "timeout": 15
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/format-on-edit.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "PostCompact": [
      {
        "matcher": "auto|manual",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/rehydrate-after-compact.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/stop-quality-gate.sh",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

Add to `.gitignore`:

```
.claude/settings.local.json
```

Official example file: https://code.claude.com/docs/en/settings-example  
Power-user hook table: https://support.claude.com/en/articles/14554000-claude-code-power-user-tips

### Step 3.3 — Write a short CLAUDE.md

Keep it under ~200 lines. Official rule of thumb: https://code.claude.com/docs/en/features-overview

Minimum sections for multi-day SDLC:

1. What this repo is and the one-sentence goal  
2. Build / test / lint / typecheck commands (copy-paste exact)  
3. Directory map (“source of truth lives in X, generated junk in Y”)  
4. Hard constraints (never edit generated files, never touch `.env`)  
5. Compaction rules (see above)  
6. Definition of done for a change  

Move playbooks (release, migrate, debug flaky CI) into **skills**, not CLAUDE.md.

---

## 4. Hooks: the reliability layer

Official contract: https://code.claude.com/docs/en/hooks

**Handler types:** `command`, `http`, `prompt`, `agent`, `mcp_tool`  
**Block signal:** exit code `2` (stderr is the reason) or JSON `decision: "block"`  
**Inject context:** stdout JSON `additionalContext` (or plain text on `SessionStart` / `UserPromptSubmit`)

### Events that matter for hour-to-multi-day runs

| Event | Fires | Can block? | Use for long AI-SWE |
|---|---|---|---|
| `SessionStart` | startup / resume / clear / compact / fork | limited | Inject git branch, open PR, failing tests, current milestone. Persist env via `$CLAUDE_ENV_FILE`. |
| `UserPromptSubmit` | every user turn | yes | Secret-scan the prompt. Reject pasted keys. Stamp the goal. |
| `PreToolUse` | before a tool | **yes** | Block `rm -rf`, force-push, prod deploys. Optionally rewrite `git status` → `rtk git status`. |
| `PostToolUse` | after success | no | Format / lint the exact file just written. |
| `PostToolUseFailure` | after failure | no | Capture the raw error for later; don’t dump 10k-line logs into chat. |
| `PreCompact` / `PostCompact` | before/after compact | Pre can block | Write a durable handoff file; re-inject CLAUDE.md highlights after compact. |
| `Stop` | agent thinks it is done | **yes** | Quality gate: if tests fail, return `decision: block` + reason so it keeps going. |
| `SessionEnd` | session closes | no | Snapshot state, notify Slack, archive the plan. |
| `ConfigChange` | settings/skills file changed live | yes | Audit unexpected hook/permission edits mid-run. |

### Concrete hook scripts

**`session-start.sh`** — cheap, deterministic context (do not dump `git log`):

```bash
#!/usr/bin/env bash
set -euo pipefail
{
  echo "branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo none)"
  echo "dirty=$(git status --porcelain | wc -l | tr -d ' ')"
  echo "head=$(git rev-parse --short HEAD 2>/dev/null || echo none)"
  echo "date=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
} 
# SessionStart stdout is added to context. Keep it tiny.
```

**`block-rm.sh`** — PreToolUse, exit 2 to block:

```bash
#!/usr/bin/env bash
# stdin is the hook JSON payload
cmd=$(jq -r '.tool_input.command // empty')
if echo "$cmd" | grep -Eq 'rm[[:space:]]+(-[a-zA-Z]*f[a-zA-Z]*|--force).*[[:space:]]/($| )'; then
  echo "Blocked catastrophic rm" >&2
  exit 2
fi
exit 0
```

**`format-on-edit.sh`**:

```bash
#!/usr/bin/env bash
set -euo pipefail
file=$(jq -r '.tool_input.file_path // empty')
[[ -z "$file" || ! -f "$file" ]] && exit 0
case "$file" in
  *.ts|*.tsx|*.js|*.jsx) npx prettier --write "$file" >/dev/null 2>&1 || true ;;
  *.py) (ruff format "$file" || true) >/dev/null 2>&1 ;;
  *.rs) (rustfmt "$file" || true) >/dev/null 2>&1 ;;
esac
exit 0
```

**`rehydrate-after-compact.sh`** — PostCompact is the long-session superpower:

```bash
#!/usr/bin/env bash
set -euo pipefail
# Print a short, durable reminder that survives the new window.
# Officially, project-root CLAUDE.md and MEMORY.md are re-injected from disk,
# but conversation-only decisions are not. Persist them yourself.
handoff="$CLAUDE_PROJECT_DIR/.claude/HANDOFF.md"
if [[ -f "$handoff" ]]; then
  echo "POST-COMPACT HANDOFF:"
  head -n 80 "$handoff"
fi
```

Maintain `.claude/HANDOFF.md` yourself (or via a Stop hook) with: current goal, files in play, last green test command, next action.

**`stop-quality-gate.sh`** — only block when you have a fast check:

```bash
#!/usr/bin/env bash
# If a cheap marker says work is unfinished, block the stop.
if [[ -f "$CLAUDE_PROJECT_DIR/.claude/KEEP_GOING" ]]; then
  echo '{"decision":"block","reason":"KEEP_GOING flag present; finish the current milestone."}'
  exit 0
fi
exit 0
```

### Hook hygiene (people get this wrong)

- Test the command **outside** Claude first. A broken PreToolUse hook that exits 2 on every Bash call freezes the agent.
- Prefer project scripts under `${CLAUDE_PROJECT_DIR}/.claude/hooks/` so teammates get the same gates.
- Use `"async": true` for slow PostToolUse tests so the agent is not blocked.
- Interactive sessions withhold hooks until workspace trust is accepted. Headless `claude -p` treats the folder as trusted — review `.claude/` before scripting over a stranger’s repo, or pass `--settings '{"disableAllHooks": true}'`.
- Cloud / web sessions do **not** read your `~/.claude/settings.json`. Team hooks must live in the repo or in managed settings.

---

## 5. Skills: auto-fire vs manual-only

Official: https://code.claude.com/docs/en/skills

### Where they live

| Scope | Path |
|---|---|
| User | `~/.claude/skills/<name>/SKILL.md` |
| Project | `.claude/skills/<name>/SKILL.md` |
| Plugin | `<plugin>/skills/<name>/SKILL.md` |
| Bundled | shipped with CLI (`/doctor`, `/code-review`, `/debug`, `/loop`, `/verify`, …) |

### Invocation controls (the two flags that matter)

| Frontmatter | You `/name` | Claude auto-loads | Description always in context? | Use when |
|---|---|---|---|---|
| default | yes | yes | yes | Repeatable workflows Claude should notice |
| `disable-model-invocation: true` | yes | no | **no** (saves tokens) | Side effects: deploy, commit, release, Slack, prod migrate |
| `user-invocable: false` | no | yes | yes | Background knowledge, not a command |

```markdown
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
context: fork
---
```

`context: fork` runs the skill in a subagent so the main multi-day thread is not flooded.

**Long-session loading rules (official):**

- Descriptions are always listed (budget ≈ 1% of the context window; per-entry cap 1,536 chars). Keep descriptions short and trigger-specific.
- Bodies load only when invoked. After compact, invoked skill bodies are re-injected with a **5,000 token / skill** and **25,000 token total** cap; oldest dropped first; truncation keeps the *start* of `SKILL.md`.
- Put the critical procedure in the first screen of `SKILL.md`. Move reference tables to `reference.md` in the same folder (progressive disclosure).

### What to make auto vs manual

**Auto-fire (default, Claude may invoke):**

- “how we write tests here”
- “API error-handling conventions”
- “how to run the local stack”
- language/style guides scoped with `user-invocable: false` if they are not slash-commands

**Manual-only (`disable-model-invocation: true`):**

- `/commit`, `/deploy`, `/release`, `/pages-prod`, `/send-slack`
- `/compact-handoff` (writes HANDOFF.md then compact)
- anything that spends money or changes shared state

**Do not auto-install 50 third-party skills.** Each description is a permanent tax on every turn. Start with 3–8 high-ROI skills. Prune anything unused for two weeks.

Bundled skills you typically leave on: `/doctor`, `/debug`, `/code-review`, `/verify`.  
Turn off bundled noise with `disableBundledSkills` or per-skill `skillOverrides` (see skills docs).

Community pack if you want a starter set rather than inventing one: https://github.com/ytrofr/claude-code-guide

---

## 6. Session, memory, and handoff protocol for multi-day work

Official memory: https://code.claude.com/docs/en/memory  
Official sessions: https://code.claude.com/docs/en/sessions  
Official 1M guidance: https://claude.com/blog/using-claude-code-session-management-and-1m-context

### What survives `/compact` (do not rely on folklore)

| Mechanism | After compact |
|---|---|
| Project-root `CLAUDE.md`, unscoped rules | Re-read from disk |
| Auto memory (`MEMORY.md` + topics) | Re-read from disk |
| Plan-mode plan file | Re-read from disk |
| Nested `CLAUDE.md` / path-scoped rules | Reloaded only when matching files are re-read |
| Recently edited files | Up to **five**, most recently modified |
| Invoked skill bodies | Re-injected with token caps above |
| Hook-injected context from earlier turns | Summarized away unless a `SessionStart` hook matches the compact source |
| Pure conversation decisions | **Gone unless you wrote them to disk** |

### Operating rhythm

1. One milestone per session. Name it: `claude --name "auth-refresh-token-rotation"`.
2. At each phase boundary (explore → implement → test → PR), run  
   `/compact focus on <milestone>; keep files, decisions, failing tests`.
3. After compact, confirm `/context` and re-open any file the five-file rehydrate missed.
4. End of day: update `.claude/HANDOFF.md` and `MEMORY.md` (`/memory`). Do not trust the transcript alone — default cleanup is 30 days unless you set 365.
5. Next morning: `claude --resume <name>` (or `--continue` for last session in this cwd).
6. If the session sat idle > ~1 hour and is >100k tokens (Pro/Max), prefer **Resume from summary** unless you know you need the raw tool traces.
7. Unrelated work → `/clear` or a new process. Do not stack a second product area onto a 6-hour debug thread.

### Subagents

Official steering post: https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more

- Fork for breadth (scan the monorepo, summarize an unfamiliar package).
- Stay in the parent for depth (the actual edit).
- Any task that will read three or more large files belongs in a child so the parent window stays clean.

---

## 7. CLI tools that extend the harness

### 7.1 RTK — Rust Token Killer (highest leverage after settings)

- Repo: https://github.com/rtk-ai/rtk  
- Docs intro: https://www.mintlify.com/rtk-ai/rtk/introduction  
- Claimed savings: 60–90% on common `git` / test / `ls` / `docker` output. A 30-minute session dropping ~150k → ~45k tokens is the published anecdote, not a guarantee.

Install (pick one):

```bash
brew install rtk
# or
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh
# or
cargo install --git https://github.com/rtk-ai/rtk
```

Wire into Claude Code:

```bash
rtk init -g          # global PreToolUse hook + RTK.md; restart claude
rtk --version
rtk gain             # savings dashboard
rtk discover         # commands you ran raw that RTK could have compressed
```

**Limitation:** the hook only rewrites **Bash** tool calls. Built-in `Read` / `Grep` / `Glob` bypass it. For huge files, prefer `offset`/`limit` reads or `rtk read`.

Enable tee-on-failure so you can recover the uncompressed log when a test fails (`~/.config/rtk/config.toml`).

### 7.2 ContextZip (output + session-archive compression)

- Repo: https://github.com/jee599/contextzip  
- Install: `npx contextzip` or `brew install jee599/tap/contextzip`  
- Useful when you want compression without RTK’s rewrite hook, or to compact stored session archives (v0.2+ roadmap / partial support).

### 7.3 Backend router / other model endpoints

Official gateway doc: https://docs.claude.com/en/docs/claude-code/llm-gateway  
LiteLLM Claude Code tutorial: https://docs.litellm.ai/docs/tutorials/claude_non_anthropic_models  
LiteLLM auto-router: https://docs.litellm.ai/docs/tutorials/claude_code_autorouter

**Minimum env (unified Anthropic-format proxy):**

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:4000"
export ANTHROPIC_AUTH_TOKEN="sk-litellm-..."
export ANTHROPIC_API_KEY=""          # empty so the CLI does not bypass the gateway
export ANTHROPIC_MODEL="claude-sonnet-5"
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1   # populate /model from GET /v1/models
```

Rotating keys instead of a static token — `settings.json`:

```json
{ "apiKeyHelper": "~/bin/get-litellm-key.sh" }
```

```bash
export CLAUDE_CODE_API_KEY_HELPER_TTL_MS=3600000
```

`ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_API_KEY` beat `apiKeyHelper`. Unset them if the helper should win.

**Auto-router pattern (cheap model for glue, expensive model for hard reasoning):** LiteLLM `auto_router/complexity_router` exposed as a single name Claude will accept, then `export ANTHROPIC_MODEL=claude-auto`. The router name *and* every leaf model must be on the org allowlist or `/model` greys it out.

**Long-session warning:** if the gateway alias hides the real context window, Claude Code will compact at the wrong time. Official fix: “Correct the window for a gateway or custom model ID” on https://code.claude.com/docs/en/context-window

Bedrock / Vertex pass-through variants are documented on the same gateway page (`CLAUDE_CODE_USE_BEDROCK=1`, `CLAUDE_CODE_SKIP_BEDROCK_AUTH=1`, etc.).

### 7.4 Hardened default config

Trail of Bits template (privacy env, deny credentials, `cleanupPeriodDays: 365`, block `rm -rf` + push-to-main):  
https://github.com/trailofbits/claude-code-config

Do **not** set `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` if you still want auto-updates; it is an umbrella kill switch. Prefer the three narrower flags in that repo (`DISABLE_TELEMETRY`, `DISABLE_ERROR_REPORTING`, `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY`).

### 7.5 Optional: tmux-driven compact

If you run unattended overnight sessions and want Claude to compact itself:  
https://github.com/sure-scale/claude-code-auto-compact

Treat this as advanced. A bad unattended compact is worse than a late official auto-compact.

---

## 8. Codex CLI — parallel playbook

Codex is the OpenAI counterpart. Concepts map cleanly; file formats do not.

| Claude Code | Codex |
|---|---|
| `CLAUDE.md` | `AGENTS.md` (and `AGENTS.override.md`) |
| `~/.claude/settings.json` | `~/.codex/config.toml` |
| `.claude/settings.json` hooks | `.codex/hooks.json` or `[[hooks.*]]` in TOML |
| `/compact` | compact events + `PreCompact` / `PostCompact` hooks |
| Skills in `.claude/skills` | Skills in `~/.codex/skills` and `.agents/skills` |
| Multi-day persistence | **`/goal`** (first-class, hours to 100h-class runs) |

### Step 8.1 — `~/.codex/config.toml` skeleton

```toml
model = "gpt-5.4"
model_reasoning_effort = "high"
approval_policy = "on-request"
sandbox_mode = "workspace-write"

[features]
codex_hooks = true

[history]
persistence = "save-all"
max_bytes = 104857600

[sandbox_workspace_write]
network_access = false
writable_roots = []

[model_providers.proxy]
name = "LLM proxy"
base_url = "http://127.0.0.1:4000/v1"
env_key = "OPENAI_API_KEY"
```

One-off overrides:

```bash
codex --model gpt-5.4
codex --profile deep
codex --config model='"gpt-5.4"'
```

Docs: https://developers.openai.com/codex/config-advanced

### Step 8.2 — Hooks

Enable the feature flag (`codex_hooks = true`).  
Files: `~/.codex/hooks.json` and/or `<repo>/.codex/hooks.json` (project hooks require trust).

Events (same names as Claude Code, by design):  
`SessionStart`, `SessionEnd`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PreCompact`, `PostCompact`, `Stop`, `SubagentStart`, `SubagentStop`, `PermissionRequest`.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.codex/hooks/session_start.py",
            "statusMessage": "Loading session notes",
            "timeout": 30
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.codex/hooks/pre_tool_use_policy.py",
            "statusMessage": "Checking Bash command",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

JSON key for timeout is `timeout` (seconds; default 600).  
`/hooks` inside the TUI is how you trust or disable non-managed hooks.

Official: https://developers.openai.com/codex/hooks/  
Portability note: several third-party catalogs (e.g. HookStack) can emit the same hook pack for Claude Code *and* Codex: https://www.hookstack.app/guides/openai-codex-hooks

### Step 8.3 — Multi-day execution: `/goal`

`/goal` is the Codex-native answer to “keep working for hours or days toward a milestone, pause/resume, steer without killing the loop.” It exists in the app, IDE extension, and CLI.  
Command reference: https://developers.openai.com/codex/cli/slash-commands  
Demo: https://youtu.be/rgh0hMYPcd0

Operational pattern:

1. Write a crisp, testable goal (not “improve the codebase”).  
2. `/goal` it. Let Codex run.  
3. Steer with follow-up messages; use side chats for questions so you don’t derail the main loop.  
4. `/goal` pause before you close the laptop; resume later.  
5. Watch disk. Multi-day Codex runs have produced large `/tmp` worktree copies when lifecycle settings failed — inspect `worktree-keep-count` and clean `/tmp` if you orchestrate many parallel runs. (Issue class: https://github.com/openai/codex/issues/35383)

### Step 8.4 — History vs rollouts (easy to get wrong)

| Path | What it is | Retention |
|---|---|---|
| `~/.codex/history.jsonl` | Prompt text only | `[history]` in config.toml (`persistence`, `max_bytes`) |
| `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` | **Full** transcript | No documented age-based auto-delete as of this writing |
| `~/.codex/memories/` | Async consolidation after hours of idle | Unused memories pruned on a ~30-day recall window |

Setting `[history] persistence = "none"` does **not** stop rollout files.  
Layout write-up: https://allaboutcoding.ghinda.com/where-ai-coding-clis-store-session-logs/

If you want 365-day *full* history on Codex, you must keep the rollout tree yourself (backup `~/.codex/sessions`) and optionally run a janitor for the rest:

- https://github.com/zzy0222/codex-session-janitor  
- https://github.com/gabrielhamalwa/codex-cliner  
- https://github.com/NathanZane/codex-assistant  

### Step 8.5 — AGENTS.md

Load order (later = more specific): user `AGENTS.md` / `AGENTS.override.md`, then each directory from git root to cwd, subject to `project_doc_max_bytes` (default 32 KiB).  
https://openai.com/index/unrolling-the-codex-agent-loop/

Keep it short for the same reason as CLAUDE.md. Put procedures in skills.

---

## 9. Decision matrix (print this)

| Need | Claude Code | Codex | Do not use |
|---|---|---|---|
| Always-on project facts | `CLAUDE.md` (<200 lines) | `AGENTS.md` | Pasting the same briefing every morning |
| Repeatable procedure Claude may start | Skill, default flags | Skill / custom prompt | A 2,000-line CLAUDE.md |
| Procedure with side effects | Skill + `disable-model-invocation: true` | Manual slash / gated hook | Auto-fire deploy skills |
| Must happen every time | Hook (`PostToolUse`, `Stop`) | Hook (`codex_hooks = true`) | “Please remember to format” |
| Survive compact | Disk files + `PostCompact` rehydrate | Disk files + `PostCompact` + memories | Chat-only decisions |
| Keep transcripts 1 year | `"cleanupPeriodDays": 365` | Backup `sessions/` (no official 365-day knob) | Assuming default retention |
| Compact earlier than default | `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` and/or `/autocompact 500k` | `PreCompact` hook + manual compact | Waiting for the red bar |
| Multi-day single objective | Named session + HANDOFF.md + `/resume` | **`/goal`** | One immortal 1M-token thread for mixed work |
| Cut token burn | RTK hook + `/context` hygiene | Same idea; compress Bash; janitor rollouts | Reading 5k-line files whole |
| Other model backends | `ANTHROPIC_BASE_URL` + LiteLLM | `[model_providers]` in config.toml | Pointing at a gateway without correcting the context window |
| Team alignment | Commit `.claude/settings.json` + hooks + skills | Commit `.codex/hooks.json` + `AGENTS.md` | Each engineer’s snowflake `~` config |

---

## 10. Copy-paste day-1 checklist

Claude Code:

- [ ] `claude` installed and `claude --version` current  
- [ ] `~/.claude/settings.json` with `cleanupPeriodDays: 365`, deny on secrets, allow on safe git/test  
- [ ] `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` (or `/autocompact` after you measure your model’s window)  
- [ ] Repo `CLAUDE.md` ≤ 200 lines + compaction rules  
- [ ] `.claude/settings.json` committed; `.claude/settings.local.json` gitignored  
- [ ] Hooks: PreToolUse deny-catastrophic, PostToolUse format, PostCompact rehydrate, Stop gate  
- [ ] 3–8 skills only; side-effect skills set `disable-model-invocation: true`  
- [ ] `rtk init -g` and `rtk gain` after one real session  
- [ ] End-of-day HANDOFF.md + `/memory` habit  
- [ ] Named sessions: `claude --name milestone-x`

Codex:

- [ ] `~/.codex/config.toml` with `[features] codex_hooks = true`  
- [ ] `AGENTS.md` at repo root  
- [ ] `~/.codex/hooks.json` trusted via `/hooks`  
- [ ] Use `/goal` for anything expected to outlive one sitting  
- [ ] Backup policy for `~/.codex/sessions/` (365-day intent)  
- [ ] Disk watch on worktrees / `/tmp` during multi-day orchestration  

---

## 11. Known pitfalls (2026)

1. **`disable-model-invocation` historically had a bug** where even user `/skill` invocation failed (GitHub issue #38969 on `anthropics/claude-code`). If `/deploy` errors with “cannot be used with Skill tool due to disable-model-invocation”, upgrade the CLI before relying on the flag.  
2. **`autoCompact: false` has been reported ignored** in some builds. Prefer threshold override + manual `/compact`, not “disable compact and pray.”  
3. **Gateway aliases lie about window size.** Compaction then fires too early or too late. Correct the window per official model-config docs.  
4. **Codex `[history]` is not the transcript.** Rollouts keep growing with no official TTL.  
5. **Headless runs execute project hooks without the trust dialog.** Audit before `claude -p` on untrusted trees.  
6. **Skill-description bloat is silent.** Fifty third-party skills can cost more than they save on a 12-hour session.  
7. **Post-compact file rehydrate is only five files.** Anything else must be in CLAUDE.md, MEMORY.md, HANDOFF.md, or a PostCompact hook.

---

## 12. Suggested next research (for a follow-up pass)

- Exact per-model default auto-compact thresholds on current Sonnet 5 / Opus 4.6 / Fable 5 1M SKUs (they move).  
- Whether `cleanupPeriodDays` now defaults to “unset / never” in some channels; official sessions page still documents **30**.  
- Codex native session-TTL if/when `[sessions.cleanup]` ships.  
- Comparative quality of PostCompact hook rehydrate vs `/compact` steering text vs MEMORY.md.  
- RTK vs ContextZip vs teaching the model to pass `head`/`rg` flags, measured on *your* monorepo.

---

*End of report. Sources are linked inline. Re-verify keys against https://code.claude.com/docs/en/settings-reference and https://developers.openai.com/codex/config-advanced before rolling into a regulated environment — both CLIs still change flag names between minor versions.*
