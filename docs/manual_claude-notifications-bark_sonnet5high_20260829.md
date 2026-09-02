# Claude Code → Bark → iPhone: HITL Notification Manual

How to get an audible, locked-screen iPhone notification the instant a
Claude Code CLI session (running in Ghostty, in a zsh + oh-my-zsh +
Powerlevel10k + `uv` venv shell on macOS) stops and needs human input —
a question, a permission prompt, or a stalled quota-resume.

Built and acceptance-tested on: MacBook Pro M4, macOS, iPhone 15 Pro,
Claude Code CLI 2.1.251, 29 Aug 2026. Derived from and narrower in scope
than `manual_hitl-remote-claude-code-iphone_v2_sol56xhigh_20260828.md`,
which covers the full four-plane remote-HITL stack (Remote Control,
Tailscale/SSH recovery, tmux, state artifacts). This manual documents
**only the attention-plane pager** — the piece that was actually
implemented and tested in this session.

---

## 1. Overview

Claude Code sometimes needs you: it asks a question, it wants permission
to run something, or an MCP tool is waiting on a form. If you're not
staring at the terminal, that moment passes silently. The fix isn't to
watch harder — it's to make the *process itself* push a notification to
your phone the instant it happens.

**Bark** is a free, open-source iOS app that exposes a personal push
endpoint: you `POST` JSON to a URL, Bark delivers it to your phone. No
account, no server to run, no cost.

**Claude Code hooks** are shell commands that Claude Code's own
CLI runs automatically at defined lifecycle events (before a tool runs,
when a permission prompt appears, on certain notification types). We
attach a hook script to the events that mean "a human is needed," and
that script fires a Bark push.

The chain, end to end:

```
Claude Code CLI hits a decision point
        │
        │  (Claude Code invokes the hook, piping JSON to stdin)
        ▼
~/.claude/hooks/hitl-pager.sh
        │
        │  (parses event JSON with jq, builds a title+body)
        ▼
curl → https://api.day.app/<YOUR_DEVICE_KEY>
        │
        │  (Bark's relay, Apple Push Notification service)
        ▼
Bark app on iPhone → banner + sound + lock-screen alert
```

Nothing about your code, diffs, or file contents is sent — only an event
label and a short truncated question string (see §7, privacy).

---

## 2. Background: why hooks, and why not just "Claude's own push"

Claude Code / the Claude iOS app already has a native push channel
(Remote Control). It's convenient but has two structural weaknesses for
this specific job:

1. **It's all-or-nothing.** You get two toggles (`Push when Claude
   decides`, `Push when actions required`) — no way to pick exactly which
   event types matter to you, and historically some builds have
   delivered push with no sound/vibration (a real reported iOS issue —
   see `manual_hitl-remote-claude-code-iphone_v2_sol56xhigh_20260828.md`
   §6, issue #53709).
2. **It's coupled to Remote Control being connected and working.** If the
   mobile UI has a bug, or Remote Control disconnects, your only doorbell
   goes with it.

A **hook-driven, independent pager** doesn't depend on Remote Control
being connected, doesn't depend on the Claude iOS app's push reliability,
and lets you choose exactly which events page you (and which don't —
notably, we deliberately exclude `idle_prompt`, which just means "60
seconds passed since Claude finished talking and you didn't type
anything." That is not "Claude is blocked," and treating it as such
produces constant false pages).

### Why `async: true` matters

Claude Code hook commands **block the tool call by default** — the tool
doesn't proceed until the hook script exits. Marking the hook
`"async": true` turns it into a fire-and-forget background call: Claude
Code doesn't wait for `curl` to finish, and the hook has no ability to
approve/deny/alter the tool call. That's exactly the behavior you want
for a pager — it should announce, never gate.

---

## 3. Prerequisites

| Requirement | Check command | What you should see |
|---|---|---|
| Claude Code CLI installed | `claude --version` | e.g. `2.1.251 (Claude Code)` |
| `jq` installed | `which jq` | a path, e.g. `/opt/homebrew/bin/jq` |
| `curl` installed | `which curl` | macOS ships this by default |
| Bark app on iPhone | — | Install from the App Store: search "Bark" (developer: Fin) |

If `jq` is missing: `brew install jq` (this manual assumes Homebrew is
already installed, consistent with the `oh-my-zsh` + `uv` shell setup).

Your shell customizations (oh-my-zsh, Powerlevel10k, `uv` venv
activation) don't interact with this setup at all — the hook script runs
via `bash`, invoked directly by the Claude Code CLI process, not through
your interactive zsh. Prompt themes, venv auto-activation, and aliases
have no effect on it.

---

## 4. Step-by-step setup

### Step 1 — Install Bark and get your device key

1. Install **Bark** from the iOS App Store.
2. Open it. On first launch it shows your personal push URL, shaped like:

   ```
   https://api.day.app/XXXXXXXXXXXXXXXXXXXXXXX
   ```

3. The trailing segment (`XXXXXXXXXXXXXXXXXXXXXXX`) is your **device
   key** — a bearer secret. Anyone with this URL can push arbitrary
   notifications to your phone. Treat it like a password:
   - Never commit it to git.
   - Never paste it into a shared/logged chat if you can avoid it.
   - Store it only in a file with `600` permissions (below).

Bark project and docs: https://github.com/Finb/Bark

### Step 2 — Create the pager config directory

```bash
mkdir -p "$HOME/.config/claude-hitl" "$HOME/.claude/hooks"
chmod 700 "$HOME/.config/claude-hitl" "$HOME/.claude/hooks"
```

### Step 3 — Write `pager.env`

Create `~/.config/claude-hitl/pager.env`:

```bash
# Treat this URL/device key as a secret. Never commit this file.
BARK_URL='https://api.day.app/YOUR_DEVICE_KEY'
BARK_LEVEL='timeSensitive' # set to critical only after testing
```

Replace `YOUR_DEVICE_KEY` with the key from Step 1. Then lock it down:

```bash
chmod 600 "$HOME/.config/claude-hitl/pager.env"
```

`timeSensitive` is Bark/iOS's alert level that's allowed to break through
most Focus modes and shows on the lock screen; `critical` is a stronger
level that can bypass the silent switch entirely, but requires enabling
Critical Alerts for Bark in iOS Settings and testing carefully before you
rely on it (see §6).

### Step 4 — Write the hook script

Create `~/.claude/hooks/hitl-pager.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

ENV_FILE="$HOME/.config/claude-hitl/pager.env"
[[ -r "$ENV_FILE" ]] || exit 0
# shellcheck disable=SC1090
source "$ENV_FILE"

payload="$(cat)"
event="$(jq -r '.hook_event_name // "Claude"' <<<"$payload")"
tool="$(jq -r '.tool_name // empty' <<<"$payload")"
kind="$(jq -r '.notification_type // empty' <<<"$payload")"
question="$(jq -r '.tool_input.questions[0].question // .message // empty' \
  <<<"$payload" | tr '\n' ' ' | cut -c1-180)"

title='Claude needs input'
body="${kind:-${tool:-$event}}"
[[ -n "$question" ]] && body="$body - $question"

jq -nc --arg title "$title" --arg body "$body" --arg level "${BARK_LEVEL:-timeSensitive}" \
  '{title:$title,body:$body,sound:"alarm",level:$level,group:"claude-hitl"}' \
  | curl -fsS --retry 2 --max-time 5 \
      -H 'Content-Type: application/json' -d @- "$BARK_URL" >/dev/null || true

exit 0
```

Make it executable, owner-only:

```bash
chmod 700 "$HOME/.claude/hooks/hitl-pager.sh"
```

**What this script does, line by line:**
- Reads `pager.env` for the Bark URL/level; exits silently (code 0) if
  it's missing, so a misconfigured pager never breaks Claude Code itself.
- Reads the hook's JSON payload from stdin (Claude Code pipes this in
  automatically — you never invoke this script directly in normal use).
- Extracts the event name, tool name, notification type, and — if
  present — the actual question text from an `AskUserQuestion` call,
  truncated to 180 characters.
- Builds a Bark JSON payload with `sound: "alarm"` and posts it with a
  5-second timeout and 2 retries, swallowing any network failure
  (`|| true`) so a flaky connection never crashes a hook.

### Step 5 — Self-test the script standalone

Before wiring it into Claude Code, test it directly:

```bash
printf '%s\n' '{
"hook_event_name":"PreToolUse",
"tool_name":"AskUserQuestion",
"tool_input":{"questions":[{"question":"Use Postgres or SQLite for MVP?"}]}
}' | "$HOME/.claude/hooks/hitl-pager.sh"
echo "exit code: $?"
```

Expected: exit code `0`, and a Bark notification arrives on your phone
titled "Claude needs input" with body "AskUserQuestion - Use Postgres or
SQLite for MVP?"

If nothing arrives, see §6 before proceeding to Step 6.

### Step 6 — Wire the hook into Claude Code

Edit `~/.claude/settings.json` (user-level, applies to **every** Claude
Code session on this Mac). If you already have other hooks configured —
common if you use other agent-orchestration tools — **append** to the
existing arrays; do not replace them. Each event type can hold multiple
matcher blocks, and Claude Code runs every hook that matches, not just
the first one.

Add (or merge into existing) entries:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "AskUserQuestion",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/hooks/hitl-pager.sh", "async": true }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/hooks/hitl-pager.sh", "async": true }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "permission_prompt|elicitation_dialog|elicitation_url_dialog|agent_needs_input|quota_auto_resume_stale|quota_auto_resume_disabled",
        "hooks": [
          { "type": "command", "command": "$HOME/.claude/hooks/hitl-pager.sh", "async": true }
        ]
      }
    ]
  }
}
```

**Why these three, and not `idle_prompt`:**

| Event / matcher | Fires when | Priority |
|---|---|---|
| `PreToolUse` / `AskUserQuestion` | Immediately before the question tool is processed | HIGH — primary doorbell |
| `PermissionRequest` / `*` | Immediately when Claude is about to ask permission | HIGH — primary doorbell |
| `Notification` / `permission_prompt\|elicitation_dialog\|elicitation_url_dialog\|agent_needs_input\|quota_auto_resume_stale\|quota_auto_resume_disabled` | ~6s after an unanswered permission/MCP prompt, or a stalled quota resume | HIGH fallback / prevents silent multi-hour stalls |
| `Notification` / `idle_prompt` | ~60s after Claude *finished responding* and you haven't typed | **Deliberately excluded** — this is not a blocked-state signal, just inactivity |

Save the file. **Back up first** if you're editing by hand:

```bash
cp ~/.claude/settings.json ~/.claude/settings.json.bak-$(date +%Y%m%d%H%M%S)
```

Validate it's still valid JSON before trusting it:

```bash
python3 -c "import json; json.load(open('$HOME/.claude/settings.json')); print('valid JSON')"
```

### Step 7 — iPhone-side checks

1. **Settings → Notifications → Bark**:
   - `Allow Notifications`: ON
   - `Sounds`: ON — **this is the single most commonly missed setting**;
     Bark can arrive as a silent banner even when everything else is
     configured correctly if this toggle is off.
   - Lock Screen / Banners: enabled as desired.
2. **Focus modes** (Settings → Focus): if any Focus mode is active,
   either add Bark to its allowed apps, or confirm the `timeSensitive`
   level is permitted through it. Test with Focus off first to isolate
   the variable.
3. **Physical silent switch / volume**: make sure the ringer/alert volume
   (Settings → Sounds & Haptics) isn't at zero, and the side switch isn't
   in silent mode — `timeSensitive` respects the silent switch;
   only `critical` can override it, and only once enabled.

### Step 8 — End-to-end acceptance test

Lock your phone, then re-run the Step 5 self-test (or trigger a real
`AskUserQuestion` inside an actual Claude Code session). Confirm:

- [ ] Text/banner arrives
- [ ] Sound/vibration fires, phone locked
- [ ] Works with Wi-Fi off (cellular only), if you need away-from-home
      coverage

---

## 5. Concrete example: what a real event looks like

When Claude Code calls `AskUserQuestion` mid-session, the hook receives
roughly this on stdin:

```json
{
  "hook_event_name": "PreToolUse",
  "tool_name": "AskUserQuestion",
  "tool_input": {
    "questions": [
      { "question": "Deploy to staging or production first?" }
    ]
  }
}
```

Which produces this Bark push payload:

```json
{
  "title": "Claude needs input",
  "body": "AskUserQuestion - Deploy to staging or production first?",
  "sound": "alarm",
  "level": "timeSensitive",
  "group": "claude-hitl"
}
```

Which shows up on the iPhone lock screen as:

```
┌─────────────────────────────────────┐
│ 🔔 Claude needs input                │
│ AskUserQuestion - Deploy to staging  │
│ or production first?                 │
│                              now      │
└─────────────────────────────────────┘
```

---

## 6. Likely issues and how to debug them

### Nothing arrives at all

1. **Test the script standalone first** (Step 5) — isolates "Bark/phone
   problem" from "Claude Code hook wiring problem."
2. **Check the Bark URL is correct**:
   ```bash
   source ~/.config/claude-hitl/pager.env
   curl -v "$BARK_URL" 2>&1 | tail -20
   ```
   A `200` response with valid JSON back means the endpoint is reachable
   and the key is accepted.
3. **Check file permissions** — the script silently exits 0 if
   `pager.env` isn't readable:
   ```bash
   ls -la ~/.config/claude-hitl/pager.env ~/.claude/hooks/hitl-pager.sh
   ```
   Expect `-rw-------` and `-rwx------` respectively, owned by you.
4. **Check `jq` is on PATH** for the shell Claude Code's hook runner
   uses — `which jq`. If installed via Homebrew on Apple Silicon, this is
   normally `/opt/homebrew/bin/jq`; if the hook runs in a minimal PATH
   environment, hardcode the full path to `jq` and `curl` inside the
   script.
5. **Validate settings.json**: a JSON syntax error anywhere in the file
   can silently disable all hooks. Run the `python3 -c "import json..."`
   check from Step 6 after any manual edit.

### Text arrives but no sound (the most common failure in practice)

This was the exact issue hit during initial testing of this setup. In
order of likelihood:

1. iOS Settings → Notifications → Bark → **Sounds** toggle is off.
   (This was the actual fix in this session's testing.)
2. A Focus mode is active and silencing it — check Settings → Focus.
3. Ringer/alert volume slider is at zero, or the side silent switch is
   engaged (Settings → Sounds & Haptics).
4. If you need to guarantee sound even through Focus/silent-switch, look
   at Bark's **Critical Alerts** setting inside the Bark app itself
   (separate from iOS Settings, because Apple gates this entitlement per
   app) and change `BARK_LEVEL` to `critical` in `pager.env` — but test
   this deliberately; critical alerts are meant to be rare/urgent by
   design.

### Notification arrives late or not at all over cellular

- Confirm you actually disabled Wi-Fi for the test — a Wi-Fi-connected
  test proves nothing about away-from-home reliability.
- APNs (Apple's push service, which Bark relies on) generally works over
  any data connection; if pages are consistently delayed only on
  cellular, check Settings → Cellular → Bark → make sure cellular data is
  allowed for the app.

### Hook seems to run but tool call hangs / Claude Code feels slow

- Confirm `"async": true` is present on every hook entry. Without it,
  Claude Code waits for the hook (including the `curl` call and its
  timeout) before proceeding — with a 5-second `--max-time` on a flaky
  network, that's a real, repeated delay per tool call.

### You have other tools' hooks already in `settings.json` and worry about conflicts

- Claude Code runs **every** hook whose matcher matches an event — not
  "first match wins." Multiple `"*"`-matcher blocks under the same event
  (e.g., one from another orchestration tool, one from this pager) both
  fire independently. As long as the other tool's hook doesn't also
  exit nonzero / block the tool call, there's no functional conflict —
  just multiple side effects per event, which is what you want from a
  pager.

---

## 7. Privacy notes

- The Bark device key is a bearer secret: whoever has the URL can push
  arbitrary text to your phone. Never commit `pager.env` to git; keep it
  at `chmod 600`.
- The hook only ever sends: an event/tool name, and — for
  `AskUserQuestion` — the question text truncated to 180 characters. It
  does **not** send file contents, diffs, full prompts, or credentials.
  Don't extend the script to include more without re-checking this.
- Rotate the Bark device key (regenerate in the Bark app) if you ever
  suspect `pager.env` was exposed (e.g., pasted into a shared chat log).

---

## 8. What this manual does *not* cover

This is the attention/pager plane only. It intentionally does not set up:

- **Remote Control** (answering the Claude conversation itself from your
  phone) — see `manual_hitl-remote-claude-code-iphone_v2_sol56xhigh_20260828.md`
  §6.
- **tmux persistence** (surviving disconnects) — ibid. §8.
- **Tailscale/SSH break-glass recovery** — ibid. §9.
- **Handoff/state artifacts** for multi-day context survival — ibid. §11.

Those remain valid next steps if you want the fuller stack; this manual
stands alone for "page me when Claude needs me."

---

## 9. Reference links

- Bark (iOS app + push relay): https://github.com/Finb/Bark
- Claude Code hooks reference: https://code.claude.com/docs/en/hooks
- Claude Code permissions reference: https://code.claude.com/docs/en/permissions
- `jq` (JSON processor): https://jqlang.org/
- Homebrew: https://brew.sh/

---

## 10. Files this setup created on this machine

| Path | Purpose | Permissions |
|---|---|---|
| `~/.config/claude-hitl/pager.env` | Bark URL + alert level (secret) | `600` |
| `~/.claude/hooks/hitl-pager.sh` | Hook script, parses event → Bark push | `700` |
| `~/.claude/settings.json` | Modified: added `PreToolUse`, `PermissionRequest`, `Notification` hook entries (appended, existing hooks preserved) | `644` |
