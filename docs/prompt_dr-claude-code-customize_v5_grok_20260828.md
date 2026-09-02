# Gemini Deep Research Prompt

**File:** `prompt_dr-claude-code-customize_v5_grok_20260828.md`  
**Created:** 2026-08-28  
**Purpose:** Paste the block under “PROMPT TO PASTE” into Google Gemini Deep Research (or Gemini Advanced Deep Research) as a single research brief. Do not split it across turns.

**How to run**

1. Open Gemini Deep Research (Google AI / Gemini Advanced). Product entry points change; as of 2026 search for “Gemini Deep Research” in Gemini.
2. Start a **Deep Research** task (not a normal chat).
3. Paste the entire prompt below unchanged.
4. Attach, if the UI allows: this file and `report_claude-code-customize_v5_grok_20260828.md` as prior art to refine, not to rubber-stamp.
5. After the report generates, export/download Gemini’s document and keep it next to these two files.

---

## PROMPT TO PASTE

```
You are conducting a Deep Research investigation with a hard as-of date of 28 August 2026.

GOAL
Produce a comprehensive, source-cited, implementation-ready research report on current best practices for customizing Anthropic Claude Code CLI and OpenAI Codex CLI specifically for LONG-RUNNING (hours to multi-day) complex AI-SWE, AI-SDLC, and AI-assisted coding sessions. The output must be concrete enough that a staff engineer can implement it the same day: exact file paths, exact JSON/TOML/YAML keys, copy-paste configs, hook scripts, skill frontmatter, environment variables, CLI flags, and full URLs.

This is not a beginner “how to install Claude Code” guide. Assume the reader already uses the CLIs daily. Focus on reliability, repeatability, context survival across compaction/resume, token efficiency, safety gates, and multi-day operational rhythm.

PRODUCTS IN SCOPE
- Anthropic Claude Code CLI (primary).
- OpenAI Codex CLI (secondary, treat as a parallel harness with different file formats).
- Closely related surfaces only when they change CLI behavior: Claude Code on the web, Codex app / IDE extension, Agent SDK, plugins, MCP, LLM gateways.

OUT OF SCOPE
- Generic prompt-engineering advice disconnected from these CLIs.
- IDE-only tools (Cursor, Windsurf, Copilot Chat) except where a hook/skill pack is shared.
- Building a new coding agent from scratch.

RESEARCH QUESTIONS (answer all)

A. Configuration surface
1. What is the current settings/config hierarchy and precedence for Claude Code (managed, --settings, settings.local.json, settings.json, ~/.claude/settings.json) and for Codex (~/.codex/config.toml, profiles, project .codex/, trust model)?
2. What is the official schema (as of Aug 2026) for the keys that matter in long sessions:
   - Claude Code: cleanupPeriodDays (confirm default: 30 or unset?), autoCompact / autoCompactEnabled / autoCompactWindow, CLAUDE_AUTOCOMPACT_PCT_OVERRIDE, /autocompact, autoMemoryEnabled, model, effortLevel, permissions allow/ask/deny, hooks, env, statusLine, sandbox, apiKeyHelper, disableAllHooks, skillOverrides, disableBundledSkills.
   - Codex: [history], rollout/session storage, [features] codex_hooks, [memories], model_reasoning_effort, sandbox_mode, approval_policy, [model_providers], worktree-keep-count, any session TTL if it now exists.
3. Confirm transcript locations and retention:
   - Claude: ~/.claude/projects/<slug>/<id>.jsonl, CLAUDE_CONFIG_DIR, CLAUDE_CODE_PROJECT_DIR_NAME, CLAUDE_CODE_SKIP_PROMPT_HISTORY, --no-session-persistence.
   - Codex: ~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl vs history.jsonl vs memories/. Does [history] persistence=none stop rollouts? Is there now a native retention_days?

B. Compaction, context, and multi-day continuity
4. What officially survives /compact and auto-compact in Claude Code (CLAUDE.md, MEMORY.md, plans, skills caps 5k/25k, five-file reread, SessionStart-on-compact, nested rules)? Cite the context-window doc table.
5. What are current default auto-compact thresholds per model family, including 1M-context SKUs (Sonnet 5, Opus 4.6+, Fable 5)? How should a practitioner set an earlier trigger (50% vs 70% vs absolute token /autocompact 500k) for multi-hour work?
6. What is the recommended operating rhythm: compact at task boundaries vs crisis; /clear vs /compact vs /rewind summarize; resume-from-summary dialog after idle >1h and >100k tokens; named sessions; HANDOFF.md patterns; subagent isolation?
7. Codex equivalent: PreCompact/PostCompact hooks, /goal for hours-to-days persistence, pause/resume, side chats, memory consolidation idle window (historically ~6h) and prune windows.

C. Hooks as deterministic control
8. Enumerate current hook events for both CLIs. For each event used in long runs (SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, PostToolUseFailure, PreCompact, PostCompact, Stop, SessionEnd, ConfigChange, PermissionRequest, Subagent*), document: matcher, blocking contract (exit 2 vs JSON decision), payload, handler types (command/http/prompt/agent/mcp_tool), timeout defaults.
9. Give production hook recipes: block catastrophic bash, format-on-edit, secret-scan prompts, PostCompact rehydrate, Stop quality-gate that prevents premature “done”, SessionStart cheap git snapshot, async test hooks.
10. Document trust/headless pitfalls: workspace trust dialog vs claude -p executing repo hooks; cloud sessions ignoring ~/.claude; disableAllHooks; managed-hooks-only.

D. Skills / commands / agents
11. Current SKILL.md frontmatter: disable-model-invocation, user-invocable, context:fork, allowed-tools, model, hooks-in-frontmatter. Exact matrix of who can invoke and whether the description stays in context.
12. Which skills should be manual-only vs auto-fire vs Claude-only background knowledge for AI-SDLC (commit/deploy/release vs style guides vs debug playbooks)?
13. Bundled Claude Code skills list and how to disable individuals vs all. Plugin vs repo vs user install. Description-budget tax (1% window, 1536 char cap) and post-compact skill body caps.
14. Known bugs to verify still open or fixed (e.g. disable-model-invocation blocking user slash commands, issue anthropics/claude-code#38969).

E. Token and cost control tools
15. RTK (Rust Token Killer): current install methods, rtk init -g hook behavior, what it can and cannot intercept (Bash vs Read/Grep/Glob), measured savings, config/tee-on-failure, GitHub URL.
16. ContextZip and any other serious token-filter CLIs. Compare.
17. Status line / /context hygiene practices that actually change outcomes.

F. Gateways and multi-model routing
18. Official way to point Claude Code at a non-Anthropic or proxied endpoint: ANTHROPIC_BASE_URL, ANTHROPIC_AUTH_TOKEN, empty ANTHROPIC_API_KEY, CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY, apiKeyHelper + CLAUDE_CODE_API_KEY_HELPER_TTL_MS, Bedrock/Vertex pass-through vars.
19. LiteLLM (and any other maintained gateway) auto-router patterns for cheap-vs-strong model tiers. How to correct context-window size for gateway aliases so autocompact is not wrong.
20. Codex [model_providers] equivalent.

G. Safety, teams, and SDLC fit
21. Permission allow/ask/deny design that reduces approval fatigue without disable-all-permissions.
22. What belongs in git (.claude/settings.json, hooks, skills, CLAUDE.md, AGENTS.md) vs gitignore (settings.local.json, secrets).
23. How these customizations map onto a real SDLC: issue → plan → implement → test → PR → deploy across multiple days and multiple engineers.

H. Synthesis
24. Produce a single decision matrix: need → Claude mechanism → Codex mechanism → anti-pattern.
25. Produce a day-1 checklist and a day-7 hardening checklist.
26. List settings/flags that changed name or default in 2026 so implementers do not copy stale blog posts.

SOURCE PRIORITY (mandatory)
Prefer primary sources, in this order:
1. Official live docs:
   - https://code.claude.com/docs/en/settings-reference
   - https://docs.claude.com/en/docs/claude-code/settings.md
   - https://code.claude.com/docs/en/hooks
   - https://code.claude.com/docs/en/hooks-guide
   - https://code.claude.com/docs/en/skills
   - https://code.claude.com/docs/en/context-window
   - https://code.claude.com/docs/en/sessions
   - https://code.claude.com/docs/en/memory
   - https://code.claude.com/docs/en/settings-example
   - https://code.claude.com/docs/en/features-overview
   - https://docs.claude.com/en/docs/claude-code/llm-gateway
   - https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more
   - https://claude.com/blog/using-claude-code-session-management-and-1m-context
   - https://support.claude.com/en/articles/14554000-claude-code-power-user-tips
   - https://developers.openai.com/codex/hooks/
   - https://developers.openai.com/codex/config-advanced
   - https://developers.openai.com/codex/cli/slash-commands
   - https://openai.com/index/unrolling-the-codex-agent-loop/
2. Official GitHub issues only to confirm bugs/behavior (anthropics/claude-code, openai/codex).
3. High-quality config templates: https://github.com/trailofbits/claude-code-config
4. Tooling: https://github.com/rtk-ai/rtk , https://github.com/jee599/contextzip , LiteLLM Claude Code tutorials.
5. Community guides only when official docs are silent; label them as community and date them.

If sources disagree, state both, say which is official, and give the safe default.

OUTPUT FORMAT (strict)
1. Title, as-of date, 10-bullet executive summary.
2. Architecture of each CLI’s customization system (diagram in words or mermaid).
3. Numbered implementation chapters matching research questions A–G, each with:
   - What to do
   - Exact file path
   - Copy-paste snippet
   - Why it matters for sessions that last hours to days
   - Full URL of the governing source
4. Decision matrix and checklists (question H).
5. “Stale advice to ignore” section (blog claims that are no longer true).
6. Open questions / values that still move between CLI minor versions.
7. Full bibliography with URLs.

QUALITY BAR
- No filler. No “it depends” without a recommended default.
- Every config key spelled as it appears in the live schema.
- Distinguish “official default”, “official override”, and “community practice”.
- Call out destructive or irreversible settings.
- Keep CLAUDE.md / AGENTS.md guidance aligned with official size advice (short always-on files; procedures in skills; determinism in hooks).

PRIOR ART TO IMPROVE, NOT COPY BLINDLY
A Grok research pass dated 2026-08-28 produced report_claude-code-customize_v5_grok_20260828.md. Use it as a hypothesis list. Verify every key against live official docs. Correct anything stale. Add anything it missed (especially per-model compact thresholds, Codex native retention if shipped, and current bundled skill lists).
```

---

## Operator notes (not part of the paste)

- Gemini Deep Research cannot be invoked from this Grok session. Running the prompt requires a Google account with Deep Research enabled.
- After Gemini finishes, save its export as `report_dr-claude-code-customize_gemini_20260828.md` (or PDF) beside this file so the two research passes can be diffed.
- If Deep Research refuses some URLs, seed it with the official doc index pages first; they link the rest.
