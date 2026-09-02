**REMOTE HUMAN-IN-THE-LOOP CONTROL**

Multi-Day Claude Code SDLC  
from MacBook + iPhone

Revised, corrected, expanded operations manual

MacBook Pro M4 / macOS Sequoia \| iPhone 15 Pro / AT&T \| Spectrum home WAN

Ghostty or terminal host \| local repo + external SSD \| 20-60 minute HITL cadence

| **Bottom line** Use first-party Claude Code Remote Control as the decision plane; add a separate audible pager triggered by deterministic Claude Code hooks; keep a durable terminal session plus Tailscale/SSH or Moshi as the execution-plane recovery path. Do not rely on a single iOS push channel, an immortal transcript, or --dangerously-skip-permissions on an unisolated laptop. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

AS OF 28 AUGUST 2026

Target file: manual_hitl-remote-claude-code-iphone_v1_sol56xhigh_20260828.md

Research basis: supplied Grok v1 manual + supplied long-session customization research brief + fresh primary-source verification on 28 Aug 2026.

# 1. Executive verdict

The original manual correctly recognized that this is not fundamentally an SSH-uptime problem. The high-cost failure is the agent reaching a human decision checkpoint while the operator is away from the Mac. The required system is therefore a multi-plane operations stack: a decision plane for the actual Claude conversation, an independent attention/pager plane, an execution-plane recovery shell, and durable project state outside the chat transcript.

## Recommended production stack

| **Plane**   | **Default**                                             | **Why**                                                                          | **Fallback**                                                                                |
|-------------|---------------------------------------------------------|----------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| Decision    | Claude Code Remote Control + Claude iOS Code tab        | First-party, bidirectional, same local process and environment.                  | Happy or heypandax/CC Pocket for sessions intentionally started under those control planes. |
| Attention   | Claude hooks -\> Bark (free OSS) or Pushover (low-cost) | Deterministic events and audible iOS delivery independent of Claude mobile push. | Moshi agent inbox/push; ntfy for non-critical/general alerts.                               |
| Persistence | tmux baseline                                           | Officially aligned, simple, mature; process survives phone/SSH disconnect.       | Herdr for agent-aware blocked/working/done/idle visibility.                                 |
| Recovery    | Tailscale + traditional SSH/Mosh; Moshi if desired      | No router port-forwarding; gives a real terminal when Remote Control UI fails.   | Local physical access.                                                                      |
| State       | Repo artifacts + issue tracker + CONTEXT/ADR/HANDOFF    | Survives compaction, resume, client changes and session loss.                    | Exported transcript is evidence, not the sole system of record.                             |

## Ten operating rules

1.  Remote Control is the primary phone conversation surface; it is not a remote shell.

2.  Treat Claude mobile push as best-effort until you personally verify sound/vibration on the exact iPhone build; current issue history includes silent iOS pushes.

3.  Page immediately on PreToolUse:AskUserQuestion and PermissionRequest; do not misuse idle_prompt as the primary "blocked" event.

4.  Keep a separate recovery path because current open iOS issues include MCP permission dialogs that can block invisibly and permission flows that can trap the composer.

5.  Run the long-lived shell in tmux; use Herdr only if its agent-awareness materially helps multi-pane operations.

6.  Do not expose SSH on the public Spectrum router. Reach the Mac through Tailscale; on ordinary macOS Tailscale clients use regular SSH over the tailnet.

7.  Leave the MacBook open and powered for the least fragile unattended host configuration; caffeinate alone is not a contractual closed-lid guarantee.

8.  Move durable decisions into CONTEXT.md, ADRs, tracker tickets, wayfinder artifacts and end-of-day HANDOFF.md; a five-day transcript is a lossy cache after repeated compaction.

9.  Prefer sandbox + auto/targeted permission rules on the local Mac. Anthropic now explicitly says bypassPermissions is for isolated containers/VMs.

10. Acceptance-test the whole path over AT&T cellular with the phone locked before depending on it away from home.

| **Version-sensitive facts** Claude Code is changing quickly. The commands and hook types below were verified against live docs on 28 Aug 2026. Re-run claude doctor and re-check the linked docs after CLI upgrades, especially Remote Control, hooks, mobile permissions, and notification behavior. |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 2. Problem restatement, scope, and major corrections to v1

The supplied background research brief asks for implementation-ready methods for long-running, hours-to-multi-day AI-SWE sessions with reliability, repeatability, context survival, token efficiency and safety gates. This manual narrows that broader problem to remote HITL operation of an already-running local Claude Code SDLC, while retaining the same reliability and safety bar.

The supplied v1 manual assumes a MacBook Pro M4 with 24 GB unified memory, iPhone 15 Pro on AT&T, Spectrum residential WAN, Ghostty, a local/external-volume checkout, a 5-day Claude Code chat, frequent compaction, mattpocock skills and 20-60 minute decision intervals. Those operational constraints remain useful; several implementation claims required correction.

## Corrections that materially change the design

| **v1 advice / implication**                                                      | **Verified correction as of 28 Aug 2026**                                                                                                                                 | **Operational import**                                                                                    |
|----------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| Remote Control effectively described as Max-only.                                | The live page currently says "available on all plans" near the top, while its Requirements section enumerates Pro, Max, Team and Enterprise; API keys remain unsupported. | Treat live account/CLI eligibility as operative and do not tie the architecture to a Max-only assumption. |
| "Local" wording could imply the session stays entirely on the Mac.               | Execution/filesystem remain local, but while Remote Control is connected the session transcript is stored on Anthropic servers for sync/reconnect.                        | Make an explicit privacy/compliance decision before enabling RC.                                          |
| Notification matcher included agent_notification.                                | Current types include agent_needs_input and agent_completed; agent_notification is not in the current matcher list.                                                       | Use live schema, not older examples.                                                                      |
| idle_prompt treated as a decision/block signal.                                  | idle_prompt fires ~60 seconds after Claude finished responding and the user has not typed.                                                                                | It is a noisy fallback, not the primary HITL doorbell.                                                    |
| Bark / ntfy grouped as equivalent Critical Alert choices.                        | Bark documents critical alerts now; ntfy iOS critical-alert support is listed for v1.8.0 UNRELEASED, while stable iOS is v1.7.0.                                          | Prefer Bark or Pushover for critical attention today.                                                     |
| Herdr presented as required host multiplexer.                                    | Official Anthropic guidance uses tmux/screen as persistence primitives; Herdr is an optional agent-aware upgrade.                                                         | Start with fewer moving parts.                                                                            |
| Tailscale SSH implied as normal on Mac.                                          | Tailscale SSH server requires the open-source tailscale+tailscaled macOS variant; standard sandboxed macOS builds should use regular SSH over Tailscale.                  | Use the supported simple path: Remote Login + SSH key + tailnet address.                                  |
| caffeinate/"disable lid sleep" could read as sufficient for clamshell operation. | Closed-display behavior on Apple Silicon is a separate operational problem; Amphetamine exposes a specific Closed-Display Mode/Power Protect path.                        | Safest unattended baseline is lid open + power + no-sleep setting.                                        |
| Happy implied it can simply attach to an arbitrary existing vanilla process.     | Happy documents starting with happy claude / happy codex; treat it as a wrapper/control plane for sessions started under it unless tested otherwise.                      | Do not wrap the irreplaceable 5-day session mid-flight.                                                   |
| --dangerously-skip-permissions normalized as a laptop default.                   | Anthropic says bypassPermissions should only be used in isolated containers/VMs where damage is constrained.                                                              | Move routine AFK work to sandbox + auto/targeted permissions; keep dangerous bypass for isolation only.   |

# 3. Target architecture: four independent planes

The robust design deliberately avoids a single point of failure. A phone notification is not the session. A session is not the durable project memory. A remote conversation UI is not an execution shell. Each responsibility gets its own mechanism.

## Reference architecture

| **iPhone**   | **Claude app / Code tab**            | **Bark or Pushover + optional Moshi**          |
|--------------|--------------------------------------|------------------------------------------------|
|              | \| decision / steer                  | ^ audible pager / ack                          |
| **Internet** | **Anthropic Remote Control (TLS)**   | **APNs / pager service + Tailscale WireGuard** |
|              | \| bidirectional sync                | \|                                             |
| **Mac host** | **tmux -\> Claude Code process**     | **hook scripts + traditional SSH/Mosh**        |
|              | \|                                   | \|                                             |
| **State**    | **repo + tests + MCP + local tools** | **CONTEXT.md / ADRs / tracker / HANDOFF.md**   |

### Decision plane

Claude Code stays on the Mac. Remote Control connects the Claude app or claude.ai/code to that local process, and both phone and terminal can write into the same conversation. Filesystem, tools and MCP remain on the host; transcript synchronization is handled by Anthropic while Remote Control is connected. \[A1\]

### Attention plane

The native Claude push channel is convenient but not sufficient as the only doorbell. Claude Code hooks provide deterministic event edges. The key design is to fire an asynchronous pager at the instant an AskUserQuestion tool call or PermissionRequest occurs, then use Notification events for delayed/fallback classes such as MCP elicitation and sandbox network prompts. \[A2\]

### Execution plane

Tailscale plus ordinary SSH/Mosh gives a real terminal path that is independent of the Remote Control renderer. It is the recovery mechanism for mobile UI bugs, MCP dialogs, reauthentication, shell-only slash commands, failed hooks, or host diagnosis. Moshi is the strongest inexpensive native mobile cockpit if you want this path to be pleasant rather than merely available. \[T1\]\[M1\]

### State plane

A multi-day session must be restartable from artifacts. Project-root CLAUDE.md, auto-memory, plans and certain recent files survive compaction in defined ways, but the transcript is still summarized. Decisions and stage status therefore belong on disk/tracker, not only in conversational history. \[A5\]\[A6\]

# 4. Ship-today setup: the 30-minute robust baseline

This chapter is the shortest path from the v1 environment to a demonstrably reliable remote loop. It does not require Herdr, Happy, Moshi Pro, a public port, or a cloud VM.

## Step 1 - verify the host and Claude authentication

cd /path/to/project  
claude --version  
claude doctor  
claude  
\# In Claude Code, confirm workspace trust and claude.ai login if prompted.

Remote Control requires a claude.ai subscription login and direct use of api.anthropic.com. API-key-only sessions, Bedrock/Agent Platform/Foundry, enterprise Claude apps gateway, or a custom ANTHROPIC_BASE_URL do not support it. Certain telemetry/feature-flag disabling variables also prevent eligibility checks. \[A1\]

## Step 2 - put the live shell in tmux

brew install tmux jq  
cd /path/to/project  
tmux new-session -A -s project-sdlc  
\# Start Claude inside tmux for a new durable session:  
claude --remote-control "Project SDLC"

For the already-running five-day chat, do not restart just to obtain tmux. First write a handoff/snapshot. If the current process is already in a durable multiplexer, leave it. If it is not, add durability at the next clean session boundary rather than risking the only live process.

## Step 3 - attach Remote Control to an existing session

\# Inside the existing Claude Code conversation:  
/remote-control  
\# or  
/rc  
  
\# Then open /config and enable:  
\# - Enable Remote Control for all sessions (optional user-level default)  
\# - Push when Claude decides  
\# - Push when actions required

The setting equivalent is remoteControlAtStartup=true in ~/.claude/settings.json. Project/local files can force false but cannot force true for everyone who opens a repo. \[A1\]

## Step 4 - install the independent pager

mkdir -p "\$HOME/.claude/hooks" "\$HOME/.config/claude-hitl"  
chmod 700 "\$HOME/.claude/hooks" "\$HOME/.config/claude-hitl"  
\# Create pager.env and hitl-pager.sh using Chapter 5.  
chmod 600 "\$HOME/.config/claude-hitl/pager.env"  
chmod 700 "\$HOME/.claude/hooks/hitl-pager.sh"

Default recommendation: Bark if you want free/open-source custom iOS push; Pushover if a \$4.99 one-time iOS license is acceptable and repeated emergency notifications/acknowledgement are useful. \[B1\]\[P1\]

## Step 5 - configure the recovery path

- **Tailscale:** install the recommended macOS standalone app; Personal is \$0 for up to 6 users for personal use. \[T2\]

- **macOS:** System Settings -\> General -\> Sharing -\> Remote Login; restrict allowed users; use an SSH key.

- **Network:** do not create router port-forwarding for SSH. Connect to the Mac's Tailscale/MagicDNS address from the phone.

- **Client:** Moshi is the most integrated agent-focused option; another SSH/Mosh client is sufficient for break-glass access.

## Step 6 - perform the cellular acceptance test

11. Turn iPhone Wi-Fi OFF so the test uses AT&T.

12. Lock the phone and put it in a pocket.

13. Trigger a controlled AskUserQuestion in the Claude session.

14. Confirm the independent pager makes an audible alert within seconds.

15. Open Claude -\> Code and answer the question; verify the turn appears in the Mac terminal.

16. Trigger a permission request in a disposable/safe context if your permission mode uses prompts.

17. Temporarily disable Remote Control or simulate a UI failure, then connect through Tailscale/SSH and reattach to tmux.

18. Record pass/fail and do not leave home until every required path passes.

# 5. Deterministic iPhone doorbell with Claude Code hooks

The most important technical correction is event selection. A remote HITL pager should fire on events that mean "the agent is asking for a human" rather than on generic inactivity.

## Recommended event policy

| **Event / matcher**                                                | **Timing**                                                             | **Pager priority** | **Use**                                                                     |
|--------------------------------------------------------------------|------------------------------------------------------------------------|--------------------|-----------------------------------------------------------------------------|
| PreToolUse / AskUserQuestion                                       | Immediately before the question tool is processed                      | HIGH               | Primary decision-page edge. AskUserQuestion is explicitly matchable.        |
| PermissionRequest / \*                                             | Immediately when Claude is about to ask permission                     | HIGH               | Primary permission-page edge when permission prompts are used.              |
| Notification / permission_prompt                                   | ~6s after an unanswered permission prompt                              | HIGH fallback      | Catches sandbox network prompts not covered by PermissionRequest.           |
| Notification / elicitation_dialog\|elicitation_url_dialog          | ~6s after MCP elicitation waits                                        | HIGH               | MCP forms/URL requests.                                                     |
| Notification / agent_needs_input                                   | When a background session is waiting, with current version constraints | HIGH               | Useful for agent-view workflows.                                            |
| Notification / quota_auto_resume_stale\|quota_auto_resume_disabled | On failed/disabled quota resume                                        | MEDIUM             | Prevents multi-hour silent stalls.                                          |
| Notification / agent_completed                                     | Completion event                                                       | LOW                | Optional chirp; can be noisy.                                               |
| Notification / idle_prompt                                         | ~60s after Claude finished responding and user did not type            | OFF by default     | Not a reliable blocked-state detector; enable only as a low-priority audit. |

## User-level hook configuration

**~/.claude/settings.json**

{  
"remoteControlAtStartup": true,  
"hooks": {  
"PreToolUse": \[  
{  
"matcher": "AskUserQuestion",  
"hooks": \[  
{  
"type": "command",  
"command": "\$HOME/.claude/hooks/hitl-pager.sh",  
"async": true  
}  
\]  
}  
\],  
"PermissionRequest": \[  
{  
"matcher": "\*",  
"hooks": \[  
{  
"type": "command",  
"command": "\$HOME/.claude/hooks/hitl-pager.sh",  
"async": true  
}  
\]  
}  
\],  
"Notification": \[  
{  
"matcher": "permission_prompt\|elicitation_dialog\|elicitation_url_dialog\|agent_needs_input\|quota_auto_resume_stale\|quota_auto_resume_disabled",  
"hooks": \[  
{  
"type": "command",  
"command": "\$HOME/.claude/hooks/hitl-pager.sh",  
"async": true  
}  
\]  
}  
\]  
}  
}

Why async: command hooks block Claude by default; async=true turns the network call into a background side effect, which is exactly what a pager should be. An asynchronous hook cannot control the tool decision, which is desirable here. \[A2\]

## Bark pager implementation (recommended free/OSS default)

\# ~/.config/claude-hitl/pager.env  
\# Treat this URL/device key as a secret. Never commit this file.  
BARK_URL='https://api.day.app/YOUR_DEVICE_KEY'  
BARK_LEVEL='timeSensitive' \# set to critical only after testing

**~/.claude/hooks/hitl-pager.sh**

\#!/usr/bin/env bash  
set -euo pipefail  
  
ENV_FILE="\$HOME/.config/claude-hitl/pager.env"  
\[\[ -r "\$ENV_FILE" \]\] \|\| exit 0  
\# shellcheck disable=SC1090  
source "\$ENV_FILE"  
  
payload="\$(cat)"  
event="\$(jq -r '.hook_event_name // "Claude"' \<\<\<"\$payload")"  
tool="\$(jq -r '.tool_name // empty' \<\<\<"\$payload")"  
kind="\$(jq -r '.notification_type // empty' \<\<\<"\$payload")"  
question="\$(jq -r '.tool_input.questions\[0\].question // .message // empty' \\  
\<\<\<"\$payload" \| tr '\n' ' ' \| cut -c1-180)"  
  
title='Claude needs input'  
body="\${kind:-\${tool:-\$event}}"  
\[\[ -n "\$question" \]\] && body="\$body - \$question"  
  
jq -nc --arg title "\$title" --arg body "\$body" --arg level "\${BARK_LEVEL:-timeSensitive}" \\  
'{title:\$title,body:\$body,sound:"alarm",level:\$level,group:"claude-hitl"}' \\  
\| curl -fsS --retry 2 --max-time 5 \\  
-H 'Content-Type: application/json' -d @- "\$BARK_URL" \>/dev/null \|\| true  
  
exit 0

Bark documents timeSensitive and critical levels; its critical level is intended to ignore silent/DND and play sound. iOS still controls app permissions, so enable the needed notification permissions and test. \[B1\]

## Pushover alternative for acknowledged/repeating alerts

Pushover costs \$4.99 once per platform for individuals after a 30-day trial. Its emergency priority=2 repeats until acknowledged and requires retry and expire values; high priority=1 plays sound/vibrates subject to device configuration. \[P1\]\[P2\]

\# Example POST for a genuine emergency-style page. Use sparingly.  
curl -fsS https://api.pushover.net/1/messages.json \\  
-F token="\$PUSHOVER_APP_TOKEN" \\  
-F user="\$PUSHOVER_USER_KEY" \\  
-F title='Claude needs input' \\  
-F message="\$body" \\  
-F priority=2 \\  
-F retry=60 \\  
-F expire=900 \>/dev/null

| **Do not leak source code into push** Pager payloads should contain only the event class, repo/session nickname and a short decision summary. Do not send diffs, secrets, credentials, private document text, or the full prompt to a third-party push service. Keep pager credentials in a chmod 600 file outside the repository. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 6. Claude Code Remote Control: exact operating model

Remote Control is the correct primary decision surface for this use case because it binds phone/web clients to a local Claude Code process rather than starting a separate vendor VM. The local process remains the executor and retains the filesystem, MCP servers and project configuration. \[A1\]

## Ways to start / attach

| **Use case**                    | **Command**                               | **Notes**                                                                               |
|---------------------------------|-------------------------------------------|-----------------------------------------------------------------------------------------|
| Dedicated server mode           | claude remote-control                     | Serves Remote Control sessions; can spawn same-dir/worktree/session modes.              |
| Normal interactive session + RC | claude --remote-control "Name"            | Local terminal stays interactive; shorthand --rc.                                       |
| Existing running conversation   | /remote-control or /rc                    | Carries the existing conversation into Remote Control.                                  |
| Resume server-started session   | claude remote-control --continue          | Current docs: works for about four hours after server stop; version requirements apply. |
| Resume by ID                    | claude remote-control --session-id \<id\> | Current docs: v2.1.200+; also about four-hour server-stop window.                       |
| Parallel server sessions        | --spawn worktree --capacity N             | Prefer worktrees over same-dir when multiple sessions edit concurrently.                |

## Security and privacy boundary

- **Good:** the Mac opens no inbound port for Remote Control; the local process makes outbound HTTPS connections through the Anthropic API over TLS.

- **Important:** while Remote Control is connected, the session transcript - messages, responses and tool activity - is stored on Anthropic servers for device sync and reconnect.

- **Still local:** execution and filesystem access remain on the host. Remote Control does not make the iPhone a filesystem mount or shell.

- **Compliance:** organizations with incompatible data-retention settings such as Zero Data Retention cannot enable Remote Control.

- **Auth:** API keys are not supported. A custom ANTHROPIC_BASE_URL or third-party gateway prevents Remote Control because the session is no longer talking directly to api.anthropic.com.

## Push behavior

In /config enable both Push when Claude decides and Push when actions required. Native push has no per-event policy beyond those toggles, which is why the independent hook pager remains necessary. iOS Focus and Notification Summary can suppress/delay pushes. \[A1\]

## Current failure modes that justify the break-glass shell

| **Issue**                                                                                         | **Status on 28 Aug 2026**                                 | **Implication**                                                                                                       |
|---------------------------------------------------------------------------------------------------|-----------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| \#53709 - iOS RC pushes arrive without sound/vibration                                            | Closed, but real historical device report (iPhone 15 Pro) | Never assume native push is an audible pager until your exact device passes a locked-phone cellular test.             |
| \#87751 - native MCP tool-permission dialogs do not render on iPhone RC; AskUserQuestion works    | Open, Aug 18 2026                                         | A session can look idle on phone while terminal shows a blocking MCP prompt. Keep terminal access.                    |
| \#89182 - composer locked during mobile permission prompt; denial can advance into another prompt | Open, Aug 24 2026                                         | Avoid approval-heavy workflows from phone; reduce prompt count with safe permissions/sandbox and retain shell access. |

## Connection notes

- **Interactive RC:** during extended network loss the local session can keep working and Claude Code retries; server mode gives up after roughly 10 minutes offline.

- **Forwarded dialogs:** permission prompts and AskUserQuestion remain open until answered; other forwarded dialogs can expire after a default wait. Keep terminal access because some session commands are local-only.

# 7. iPhone 15 Pro configuration and alert acceptance tests

The phone is a client, not the process host. iOS can suspend the app; the Mac/tmux session must remain authoritative. Configure notifications and Focus to make the phone a reliable operator endpoint, then test real cellular delivery.

## Claude app

19. Install/update Claude; sign in with the same account and organization as the terminal session.

20. Open Code and verify the named Remote Control session appears with online status.

21. iOS Settings -\> Notifications -\> Claude: Allow Notifications ON; enable Sounds, Lock Screen and Banners as desired.

22. Claude Code /config: enable Push when Claude decides and Push when actions required.

23. If /config says no mobile is registered, open the Claude app so it refreshes its push token, then reconnect Remote Control.

## Pager app

- **Bark:** enable notifications; use timeSensitive for routine HITL, and only use critical after confirming the app/device permits it and the behavior is appropriate.

- **Pushover:** choose an alert sound and use priority 1 for high priority or priority 2 only when repeated acknowledgement-style paging is warranted.

- **Focus:** either allow the pager app through relevant Focus modes or use a tested interruption level. Notification Summary should not batch the pager.

- **AT&T path:** test with Wi-Fi disabled. A local-LAN test proves almost nothing about away-from-home reliability.

## Acceptance test matrix

| **Test**            | **Expected phone behavior**                       | **Expected Mac behavior**           | **Pass criterion**                                          |
|---------------------|---------------------------------------------------|-------------------------------------|-------------------------------------------------------------|
| AskUserQuestion     | Pager sounds; Claude Code card visible            | Terminal shows pending question     | Phone answer resumes turn within seconds.                   |
| PermissionRequest   | Pager sounds immediately; native RC may also push | Terminal shows permission prompt    | Remote response or shell answer unblocks safely.            |
| MCP elicitation     | Pager after ~6s if unanswered                     | Terminal shows elicitation form/URL | Operator knows it is blocked even if mobile renderer fails. |
| Network switch      | Phone moves Wi-Fi -\> LTE                         | Claude process unaffected           | RC reconnects or shell path remains available.              |
| RC UI failure       | Claude UI unavailable/stuck                       | tmux process continues              | Tailscale/SSH attaches and resolves block.                  |
| Phone locked 20 min | No active app required                            | Host keeps working                  | Next hook still produces audible notification.              |

# 8. Treat the MacBook as a temporary server - without pretending it is one

The MacBook Pro M4 is capable, but a laptop has failure modes a Mac mini or server does not: lid-close sleep, power transitions, Wi-Fi roaming, GUI memory pressure and external-drive disconnection. The right response is operational discipline, not more remote-control software.

## Power / sleep baseline

- **Least fragile:** power adapter connected, lid open, macOS setting to prevent automatic sleep while display is off, display allowed to sleep, and caffeinate as an additional assertion if desired.

- **Closed lid:** use Apple-supported clamshell conditions or a deliberately tested utility. Amphetamine exposes Closed-Display Mode and Drive Alive; on Apple Silicon its Power Protect feature exists because power-source changes can affect closed-display behavior. \[MAC1\]

- **Do not claim:** caffeinate -dims by itself guarantees that an Apple Silicon MacBook will remain an unattended server with the lid closed.

\# Additional keep-awake assertion while the shell is running.  
\# Treat this as one layer, not a closed-lid guarantee.  
caffeinate -dimsu

## Process persistence

\# Named durable session  
tmux new-session -A -s project-sdlc  
  
\# Useful checks  
tmux list-sessions  
tmux attach -t project-sdlc

If you prefer richer agent visibility, Herdr is a credible fast-moving upgrade: its sidebar recognizes blocked/working/done/idle states and supports Claude Code and Codex. It is not necessary for correctness and should not displace tmux unless its additional state model pays for the extra dependency. \[H1\]

## 24 GB memory budget

- **Cap concurrency:** do not equate "subagent" with a separate giant local model, but remember that browser tabs, IDEs, test servers, Docker, local databases, build caches and terminal panes can drive swap pressure.

- **Use worktrees:** for parallel editing, use separate worktrees rather than several agents writing the same checkout. Remote Control server mode now has --spawn worktree.

- **Observe:** keep Activity Monitor or memory_pressure available; close unused Electron/browser workloads before multi-hour unattended waves.

- **Fail safe:** if the host starts thrashing, reduce parallel work before adding another phone-control daemon.

## External 4 TB SSD

- **Durability:** if the active repo is on the OWC volume, a cable/power/idle unmount is equivalent to losing the project mid-session.

- **Safer placement:** keep the active git worktree and critical environment on the internal SSD when practical; use external storage for corpora, caches and large assets.

- **If external is required:** use a stable cable/power path, disable unintended drive sleep where appropriate, and include "volume mounted" in the recovery checklist.

# 9. Break-glass terminal: Tailscale + regular SSH/Mosh

Do not port-forward the Mac's SSH service on a residential router. Tailscale gives the iPhone and Mac a private WireGuard overlay and works through NAT/CGNAT. For the normal macOS Tailscale app, use ordinary SSH over the tailnet rather than assuming the Mac is a Tailscale SSH server. \[T1\]\[T3\]

## Mac setup

24. Install the recommended Tailscale standalone macOS client and sign into the same tailnet as the iPhone.

25. System Settings -\> General -\> Sharing -\> Remote Login: turn on. Limit SSH access to the intended account.

26. Generate/import an SSH key in the phone client and append only the public key to ~/.ssh/authorized_keys on the Mac.

27. From the iPhone, connect to the Mac's MagicDNS name or Tailscale IP with the ordinary SSH/Mosh client.

28. Reattach to tmux: tmux attach -t project-sdlc.

| **Tailscale SSH nuance** Tailscale documents that its SSH server component on macOS is available only on the open-source tailscale+tailscaled CLI variant. The recommended standalone/App Store macOS apps are not Tailscale SSH servers. Regular SSH over the Tailscale network works and is the simpler choice here. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## Moshi as the rich execution-plane client

Moshi is strong when you want more than emergency shell access: native chat support for Claude Code and Codex, agent-aware hooks/inbox, push notifications, diff/browser views, SSH/Mosh transport and direct transcript streaming from host to phone through an SSH-forwarded local gateway. Its current free tier includes SSH, agent monitoring and push; Pro is \$7.99/month, \$69.99/year, or \$199 lifetime (limited offer, verify before purchase). \[M1\]\[M2\]

Use Moshi when one iOS app should also drive Codex/Grok/other CLI panes, when you want Dynamic-Island/agent-inbox style workflow, or when Remote Control mobile rendering is the recurrent failure. It is an execution cockpit, not a reason to remove first-party Remote Control if Remote Control is working well.

# 10. Market landscape: ranked alternatives and when to use them

There are three distinct product classes. Do not compare them as if they were interchangeable: first-party conversation sync, third-party agent control planes, and remote terminals solve different parts of the problem.

| **Option**            | **Class / cost**                                 | **Strength**                                                           | **Important caveat**                                                                   | **Rank for this use**                            |
|-----------------------|--------------------------------------------------|------------------------------------------------------------------------|----------------------------------------------------------------------------------------|--------------------------------------------------|
| Claude Remote Control | First-party; included with eligible subscription | Same local Claude session; bidirectional phone/web/terminal control    | Transcript stored on Anthropic while connected; mobile push/UI issues exist            | 1 - primary decision plane                       |
| Bark                  | Free OSS iOS pager                               | Simple custom APNs push; timeSensitive/critical documented             | Device key is a secret; no agent chat                                                  | 1A - default pager                               |
| Pushover              | \$4.99 one-time iOS individual license           | Mature high/emergency priority; emergency repeats until acknowledged   | Third-party service; keep payload minimal                                              | 1B - strongest low-cost pager                    |
| Tailscale + SSH/tmux  | \$0 Personal for non-commercial personal use     | Vendor-neutral break-glass shell without public port                   | Requires host awake + SSH hardening                                                    | 1 - recovery plane                               |
| Moshi                 | Free terminal; optional Pro                      | Agent inbox/chat + real terminal + Mosh/Tailscale; multi-CLI           | Commercial; more moving parts than pure RC                                             | 2 - best rich recovery / multi-CLI cockpit       |
| Happy                 | MIT; free OSS                                    | Claude + Codex mobile/web with E2E encryption; mature adoption         | Docs say start with happy claude/codex; treat as wrapper for sessions started under it | 2 - best mature OSS alternative control plane    |
| heypandax/CC Pocket   | MIT; local-first OSS                             | Zero-knowledge relay, E2E, multi-agent control from mobile             | Younger ecosystem; verify iOS distribution/setup path before relying on it             | 3 - promising local-first alternative            |
| Herdr                 | OSS terminal multiplexer fork                    | Agent blocked/working/done/idle awareness; detach/reattach             | Not needed for core durability; install method currently repo script                   | 3 - optional host upgrade                        |
| cmux                  | Free/open source; iOS beta early access          | Ghostty-based integrated agent terminal/workspace; mobile connect beta | iOS app is beta/TestFlight/early access; not reliability baseline yet                  | Watch - fast-rising                              |
| ntfy                  | Open source; self-hostable                       | Excellent general notifications; cross-platform                        | iOS critical-alert support is v1.8.0 UNRELEASED as of date                             | Fallback/general alerts, not sole critical pager |

## Happy

Happy is the strongest mature free/OSS alternative if you want one mobile/web UI for Claude Code and Codex. The project is MIT-licensed, advertises end-to-end encryption, and currently documents npm install -g happy followed by happy claude or happy codex. That startup model is the key operational caveat: trial it on a disposable session, then adopt it for future sessions if it meets your same-session requirements. Do not insert it blindly around the irreplaceable five-day process. \[HP1\]

## CC Pocket

The heypandax/cc-pocket project describes itself as a local-first control plane with an end-to-end encrypted zero-knowledge relay and no content logging. It supports Claude Code and Codex and is MIT-licensed. The architecture is compelling for operators who distrust hosted transcript services, but its install/distribution path is more involved than first-party Remote Control; validate the exact iOS release and upgrade workflow before making it P0 infrastructure. \[CC1\]

## cmux

cmux is a notable fast-rising Ghostty-based open-source macOS agent terminal. Its repository states that the iOS app is in beta on TestFlight and early access is tied to the Founders Edition. That makes it strategically interesting, especially for a Ghostty-centric operator, but not the most conservative production control path today. \[C1\]

## ntfy

ntfy remains excellent for open-source/self-hosted notifications. However, its own release notes list stable iOS as v1.7.0 and critical alerts that bypass silent/DND under iOS v1.8.0 UNRELEASED. Therefore do not repeat the v1 manual's implication that current stable ntfy iOS is equivalent to Bark/Pushover for critical paging. \[N1\]

# 11. Multi-day continuity: compaction, resumes, handoffs and mattpocock skills

The broader research brief correctly treats context survival as an operational concern, not a prompt-writing trick. A remote-HITL system fails if the phone can answer but tomorrow's session no longer knows what was decided.

## What survives compaction - and what that implies

| **Mechanism**                           | **After compaction**                            | **Action for this workflow**                                                 |
|-----------------------------------------|-------------------------------------------------|------------------------------------------------------------------------------|
| Project-root CLAUDE.md / unscoped rules | Re-injected from disk                           | Keep always-on instructions short and high-value.                            |
| Auto memory                             | Re-injected                                     | Useful for preferences/learned facts, not a transaction log.                 |
| Plan-mode plan                          | Re-injected                                     | Use for active plan state, but still externalize stage decisions.            |
| Nested/path-scoped rules                | Reload when matching files are read             | Do not assume they are present immediately after compact.                    |
| Recently read/edited files              | Up to five reread, most recently modified first | Make HANDOFF/CONTEXT current before compact so they are likely to be useful. |
| Invoked skill bodies                    | Re-injected with per-skill/total caps           | Put critical instructions at top of long SKILL.md files.                     |
| Prior hook-added context                | Summarized with conversation                    | Hooks are not durable memory unless they read durable files again.           |
| SessionStart hook source=compact        | Runs and can add output after compact           | Optional place to rehydrate current ticket/handoff state.                    |

## Recommended rhythm

29. At every stage boundary or architecture decision, update the tracker/wayfinder ticket and ADR/CONTEXT before continuing.

30. Before a deliberate compact, write the current objective, branch/worktree, completed tests, decisions, one open HITL question and the next command to HANDOFF.md.

31. Run /compact with a focus instruction at task boundaries rather than waiting for crisis compaction.

32. Use /clear when moving to unrelated work; do not drag an old epic into a new task merely because the chat is "alive."

33. Use /context periodically to see actual context composition and optimize what is always loaded.

34. After \>~1 hour inactivity on Pro/Max and \>100k tokens, evaluate the Resume-from-summary dialog; choose summary when exact old detail is no longer operationally needed.

35. Delegate large searches/reads to subagents so their raw context does not pollute the main decision thread.

36. At end of day, run the handoff skill or write HANDOFF.md and ensure tracker status is authoritative before leaving the session unattended overnight.

## mattpocock skills mapping

| **Skill / pattern**           | **Remote HITL rule**                                                                                                                                                                                            |
|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| grill-me / grilling           | One decision at a time. Prefer AskUserQuestion for actual blocking choices so hooks can page deterministically; current community issue history means you should not assume every skill version always uses it. |
| grill-with-docs               | Use the CONTEXT.md/ADR artifacts as durable decision memory; they are more valuable remotely than a long interview transcript.                                                                                  |
| wayfinder                     | Its purpose is work too large for one session; keep HITL decision tickets separate from AFK implementation tickets and resolve human tickets through actual human exchange.                                     |
| to-spec / to-tickets          | Convert decisions into discrete work with explicit blockers before launching parallel AFK waves.                                                                                                                |
| implement / TDD / code-review | AFK work can proceed through tests and review gates; stop before irreversible deploy/push or a new product decision.                                                                                            |
| handoff                       | Mandatory at day boundaries, major compaction, or before migrating to another control plane such as Happy.                                                                                                      |
| wizard                        | Good for steps that truly require a human/browser/credential action; the phone pager should summon the operator rather than pretending the agent can do it.                                                     |

# 12. Remote safety: replace approval fatigue without disabling the safety boundary

The original environment runs with --dangerously-skip-permissions because the operator does not want to tap Allow/Deny repeatedly. That goal is reasonable; the mechanism is too broad for an unisolated personal laptop. Anthropic now explicitly says bypassPermissions should be used only in isolated containers or VMs where Claude Code cannot cause damage. \[A3\]

## Preferred local-Mac policy

37. Use Claude Code sandboxing for Bash where possible. In auto-allow sandbox mode, sandboxable commands can run without asking while filesystem/network boundaries are still enforced.

38. Use auto permission mode if available and appropriate, or use default/acceptEdits plus targeted allow rules for recurring safe commands.

39. Deny outbound/irreversible commands you do not want an unattended agent to perform. The official docs show a deny rule for Bash(git push \*).

40. Keep deployments, production mutations, secret changes and irreversible data operations behind explicit human gates.

41. Treat hooks as deterministic policy/telemetry, not as a substitute for OS-level isolation.

## Example settings fragment

{  
"permissions": {  
"allow": \[  
"Bash(npm run \*)",  
"Bash(git commit \*)"  
\],  
"deny": \[  
"Bash(git push \*)"  
\]  
}  
}

This exact pattern is aligned with Anthropic's documented example: allow npm scripts and git commits, refuse git push. Adapt the allow list to the project rather than using broad Bash(git \*) patterns. Deny rules take precedence over ask and allow. \[A3\]

## AFK vs HITL execution policy

| **Ticket class**                | **Agent may do unattended**                                                                           | **Must stop/page before**                                                                             |
|---------------------------------|-------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| AFK implementation              | Edit code, run unit/integration tests, local builds, static analysis, local commits if policy permits | New architecture/product decision; remote push/deploy; credential changes; destructive data migration |
| HITL design / grill / wayfinder | Read, compare options, prototype in throwaway/local scope                                             | Choosing option, changing acceptance criteria, altering architecture baseline                         |
| Review / hardening              | Read, test, lint, analyze diffs, propose patches                                                      | Publishing security-sensitive output; destructive remediation; production change                      |
| Deploy / migration              | Prepare plan, preflight, dry-run where safe                                                           | Real production mutation unless explicitly approved for that step                                     |

## If you deliberately keep bypassPermissions

| **Higher-risk compatibility mode** Do not pretend a phone pager makes bypassPermissions safe. Use an isolated worktree/container/VM, no production credentials, least-privilege tokens, deterministic destructive-command hooks where useful, frequent git snapshots, and a clear blast radius. Re-test permission/deny behavior after version changes. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 13. Failure matrix and recovery runbook

| **Symptom**                                                 | **Likely layer**                    | **Immediate action**                                                                                 | **Long-term fix**                                                                              |
|-------------------------------------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Claude phone push visible but silent                        | Native attention plane              | Use Bark/Pushover; answer in Claude app                                                              | Keep independent pager; retest after app/CLI updates                                           |
| No pager on question                                        | Hook config/script                  | From shell: inspect ~/.claude/settings.json; run hook script with sample JSON; check jq/curl/network | Add logging with no secrets; keep hook user-local                                              |
| Phone shows session idle; terminal is blocked on MCP prompt | RC mobile renderer / MCP permission | Tailscale/SSH -\> tmux; answer terminal dialog                                                       | Keep break-glass terminal; minimize MCP permission churn                                       |
| Composer locked in permission sequence                      | RC mobile permission UX             | Use terminal to deny/steer; do not keep tapping through unknown chain                                | Use sandbox/allow rules so routine work does not generate approval marathon                    |
| Remote Control disconnects, local terminal still works      | Decision plane network/auth         | Read RC status reason; /remote-control or resume path as documented                                  | Check direct api.anthropic.com, feature-flag env vars, account eligibility                     |
| Server-mode RC exited after outage                          | RC server mode                      | Restart claude remote-control; use --continue/--session-id if within supported window                | Prefer interactive RC for a single critical long conversation if local terminal remains active |
| SSH unavailable but RC works                                | Execution plane                     | Continue through RC; do not reboot host remotely unless necessary                                    | Check Tailscale, Remote Login, SSH key, firewall                                               |
| Both RC and SSH unavailable                                 | Host/network/power                  | Wait for network if likely; physical access if host asleep/crashed                                   | Open-lid powered baseline; Ethernet if available; monitor external drive                       |
| External SSD vanished                                       | Storage                             | Stop agent writes; reconnect/remount; verify repo and git status                                     | Move active worktree internal or harden cable/power/Drive Alive                                |
| Context lost after compact                                  | State plane                         | Read HANDOFF/CONTEXT/tracker; /context; restate current ticket                                       | Make artifact update mandatory before compaction/day end                                       |
| Host swapping/thrashing                                     | Resource plane                      | Stop/reduce parallel agents/apps; preserve state                                                     | Cap concurrency; internalize active repo; close browser/Electron loads                         |
| Need Codex/Grok/Gemini from phone                           | Product boundary                    | Use Moshi/Happy/CC Pocket or ordinary shell, not Claude RC                                           | Adopt multi-vendor cockpit intentionally                                                       |

## Five-minute recovery sequence

42. Check the Mac is awake/powered and the project volume is mounted.

43. From iPhone, try Claude Code Remote Control. If the prompt is visible, answer there.

44. If RC is ambiguous, open Tailscale and the SSH/Mosh client; attach to tmux.

45. Inspect the exact terminal dialog before choosing an action. Do not blindly approve a queue.

46. If context looks wrong, stop new work, read HANDOFF.md/CONTEXT.md/tracker and re-establish the current ticket.

47. Only after state is secure, repair the notification/RC layer. The work process is more valuable than the remote UI.

# 14. Acceptance criteria, day-1 checklist, day-7 hardening

The system is "done" only when its failure paths are tested. A QR code that pairs successfully is not an acceptance test.

## P0 - before leaving the house

| **Check**               | **Done when**                                                                           |
|-------------------------|-----------------------------------------------------------------------------------------|
| Remote Control attached | Same phone response appears in the live Mac conversation within seconds.                |
| Native push configured  | Both /config push toggles enabled; visual notification arrives on iPhone.               |
| Independent pager       | AskUserQuestion triggers a sound while phone is locked and Wi-Fi is off.                |
| Permission signal       | Safe test PermissionRequest pages immediately if permissions are enabled.               |
| MCP fallback            | Operator knows how to reach terminal for an invisible MCP permission dialog.            |
| Host persistence        | Disconnect phone/SSH; Claude remains alive in tmux.                                     |
| Recovery shell          | Tailscale + SSH/Mosh reaches Mac over AT&T without router port-forward.                 |
| Power                   | Mac remains awake in the exact unattended lid/power configuration for at least an hour. |
| State                   | HANDOFF.md/CONTEXT/tracker show current stage, decisions, tests and next HITL question. |

## P1 - next day

- Run one full 20-60 minute AFK wave from phone-away state and measure time-to-page and time-to-resume.

- Add a short hook log (timestamps/event type only) to diagnose missed notifications without storing prompts.

- Choose tmux vs Herdr based on observed multi-pane need; do not install Herdr merely because the prior manual ranked it.

- Replace broad permission bypass with sandbox/targeted permission rules where practical.

- Test a deliberate /compact at a clean task boundary; verify open question and current ticket remain obvious.

## P2 - by day 7

- Run a controlled Remote Control failure drill and recover entirely through Tailscale/SSH.

- Trial Happy on a throwaway repo if you want a free OSS parallel control plane for Claude + Codex.

- Trial Moshi free tier if a richer terminal/agent inbox would reduce recovery friction; buy Pro only if the free tier proves insufficient.

- Review pager payload privacy and rotate/revoke any push credentials that were accidentally logged or committed.

- Pin/record the Claude Code version used for the project and re-check hook matchers after major upgrades.

- Measure whether 50% auto-compaction is helping. The live docs support explicit /autocompact token thresholds; choose a threshold from actual /context behavior, not habit alone.

- If running concurrent remote sessions, migrate shared-directory parallel editing to worktrees.

# 15. Stale advice to ignore

48. "Remote Control is Max-only." The same live page currently says "available on all plans" but its Requirements section enumerates Pro, Max, Team and Enterprise. Treat this as a documentation inconsistency; verify live eligibility on the account/CLI.

49. "Remote Control means nothing leaves the Mac." Execution/filesystem stay local, but the connected session transcript is stored on Anthropic servers.

50. "idle_prompt means Claude is blocked." It means Claude finished responding about 60 seconds ago and you did not type.

51. "agent_notification is the current Notification matcher." Use the current matcher set; agent_needs_input/agent_completed are current names.

52. "ntfy stable iOS has Critical Alerts." ntfy documents that feature for iOS v1.8.0 UNRELEASED as of this report.

53. "Herdr is required for durability." tmux/screen are the simple baseline; Herdr is an agent-aware enhancement.

54. "Tailscale SSH works as the server on the standard Mac app." On macOS, Tailscale SSH server support is limited to the open-source tailscale+tailscaled variant; regular SSH over the tailnet is the baseline.

55. "caffeinate means you can close the lid and forget the MacBook." Treat closed-display operation as a separate tested host configuration.

56. "Happy can be dropped around any existing session with zero migration risk." Its documented happy claude/codex launch path means existing-process attachment should be proven, not assumed.

57. "--dangerously-skip-permissions is just a convenience mode for a personal Mac." Anthropic now says bypassPermissions is for isolated environments where damage is constrained.

58. "A mattpocock grilling skill will always surface its question as AskUserQuestion." Current issue history shows this should not be assumed; keep the standing CLAUDE.md rule and a lower-priority fallback pager.

59. "A five-day compacted transcript is the project record." The project record is the repo/tracker/decision artifacts; the transcript is an execution cache with useful evidence.

# 16. Copy-paste operator snippets

## Standing project instruction

**CLAUDE.md (project root; keep this section short)**

\# Remote HITL - operator is often on iPhone  
  
- When a human decision, authorization, redirect, architecture choice, or stage gate is required:  
1. Use AskUserQuestion whenever the choice can be expressed cleanly with 2-4 options.  
2. Keep the question short enough to read on a phone.  
3. Include the blast radius / trade-off in \<=5 concise lines.  
4. Wait for the human answer; do not infer or self-answer a HITL ticket.  
- Do not hide an open question inside a long status report.  
- After /compact, first restate the active ticket and the single open HITL question, if any.  
- AFK tickets: implement, test, and review within the agreed blast radius; stop before a new product/architecture decision.  
- Never push/deploy/force-reset/clean destructive state unless the current ticket explicitly authorizes that exact action and the permission policy allows it.  
- Before a deliberate compact and at end of day, update HANDOFF.md plus the tracker/wayfinder ticket.  
- HANDOFF.md must contain: objective, branch/worktree, done, in-progress, tests, decisions/ADRs, blockers, open HITL question, next command.

## HANDOFF.md template

\# HANDOFF  
Updated: YYYY-MM-DD HH:MM TZ  
  
\## Objective  
\<one sentence\>  
  
\## Current ticket / stage  
\<tracker URL or issue ID\>  
  
\## Branch / worktree / cwd  
\<branch\>  
\<path\>  
  
\## Done  
- ...  
  
\## In progress  
- ...  
  
\## Tests / checks  
- PASS: ...  
- FAIL / not run: ...  
  
\## Decisions / ADRs  
- ...  
  
\## Open HITL question  
\<exactly one blocking question, or NONE\>  
  
\## Next safe action  
\<single next command / ticket\>

## Sample hook self-test

printf '%s\n' '{  
"hook_event_name":"PreToolUse",  
"tool_name":"AskUserQuestion",  
"tool_input":{"questions":\[{"question":"Use Postgres or SQLite for MVP?"}\]}  
}' \| "\$HOME/.claude/hooks/hitl-pager.sh"

## Version / environment diagnostic

claude --version  
claude doctor  
tailscale status  
tmux list-sessions  
mount \| grep -i '/Volumes/' \|\| true  
ps aux \| grep -E '\[c\]laude\|\[t\]mux'  
\# Optional resource snapshots:  
memory_pressure  
df -h

# 17. Sources and verification notes

Source policy: primary live vendor documentation first; official GitHub repositories/issues for current behavior/bugs; product documentation/pricing for third-party tools. The supplied Grok manual is treated as prior art to verify, not as authority. Prices and beta status can change after 28 Aug 2026.

## Anthropic / Claude Code primary sources

**\[A1\] Remote Control:** [<u>https://code.claude.com/docs/en/remote-control</u>](https://code.claude.com/docs/en/remote-control)

**\[A2\] Hooks reference:** [<u>https://code.claude.com/docs/en/hooks</u>](https://code.claude.com/docs/en/hooks)

**\[A3\] Permissions:** [<u>https://code.claude.com/docs/en/permissions</u>](https://code.claude.com/docs/en/permissions)

**\[A4\] Sandboxing:** [<u>https://code.claude.com/docs/en/sandboxing</u>](https://code.claude.com/docs/en/sandboxing)

**\[A5\] Context window / compaction:** [<u>https://code.claude.com/docs/en/context-window</u>](https://code.claude.com/docs/en/context-window)

**\[A6\] Sessions / resume:** [<u>https://code.claude.com/docs/en/sessions</u>](https://code.claude.com/docs/en/sessions)

**\[A7\] Memory / CLAUDE.md:** [<u>https://code.claude.com/docs/en/memory</u>](https://code.claude.com/docs/en/memory)

**\[A8\] Mobile:** [<u>https://code.claude.com/docs/en/mobile</u>](https://code.claude.com/docs/en/mobile)

## Current Claude Code issue evidence

**\[G1\] \#53709 - RC push notifications have no sound/vibration on iOS:** [<u>https://github.com/anthropics/claude-code/issues/53709</u>](https://github.com/anthropics/claude-code/issues/53709)

**\[G2\] \#87751 - iPhone RC native MCP tool-permission dialogs never render:** [<u>https://github.com/anthropics/claude-code/issues/87751</u>](https://github.com/anthropics/claude-code/issues/87751)

**\[G3\] \#89182 - mobile permission prompt locks composer / denial advances:** [<u>https://github.com/anthropics/claude-code/issues/89182</u>](https://github.com/anthropics/claude-code/issues/89182)

## mattpocock skills

**\[S1\] mattpocock/skills README:** [<u>https://github.com/mattpocock/skills</u>](https://github.com/mattpocock/skills)

**\[S2\] wayfinder documentation:** [<u>https://github.com/mattpocock/skills/blob/main/docs/engineering/wayfinder.md</u>](https://github.com/mattpocock/skills/blob/main/docs/engineering/wayfinder.md)

**\[S3\] grill-with-docs skill:** [<u>https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md</u>](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md)

**\[S4\] Issue \#850 - grilling / AskUserQuestion behavior:** [<u>https://github.com/mattpocock/skills/issues/850</u>](https://github.com/mattpocock/skills/issues/850)

## Remote-control / pager / terminal ecosystem

**\[HP1\] Happy - MIT Claude Code/Codex mobile and web client:** [<u>https://github.com/slopus/happy</u>](https://github.com/slopus/happy)

**\[CC1\] heypandax/CC Pocket - local-first control plane:** [<u>https://github.com/heypandax/cc-pocket</u>](https://github.com/heypandax/cc-pocket)

**\[H1\] Herdr - agent-aware terminal multiplexer:** [<u>https://github.com/motionharvest/herdr</u>](https://github.com/motionharvest/herdr)

**\[C1\] cmux - Ghostty-based open-source macOS agent terminal:** [<u>https://github.com/manaflow-ai/cmux</u>](https://github.com/manaflow-ai/cmux)

**\[B1\] Bark - iOS custom push / time-sensitive / critical alerts:** [<u>https://github.com/Finb/Bark</u>](https://github.com/Finb/Bark)

**\[P1\] Pushover pricing:** [<u>https://pushover.net/pricing</u>](https://pushover.net/pricing)

**\[P2\] Pushover message API / priority behavior:** [<u>https://pushover.net/api</u>](https://pushover.net/api)

**\[N1\] ntfy release notes:** [<u>https://docs.ntfy.sh/releases/</u>](https://docs.ntfy.sh/releases/)

**\[N2\] ntfy publish docs:** [<u>https://docs.ntfy.sh/publish/</u>](https://docs.ntfy.sh/publish/)

**\[M1\] Moshi Claude Code guide:** [<u>https://getmoshi.app/guides/claude-code</u>](https://getmoshi.app/guides/claude-code)

**\[M2\] Moshi hooks / agent inbox:** [<u>https://getmoshi.app/docs/hooks</u>](https://getmoshi.app/docs/hooks)

**\[M3\] Moshi pricing:** [<u>https://getmoshi.app/pricing</u>](https://getmoshi.app/pricing)

## Network / host sources

**\[T1\] Tailscale SSH feature and macOS server limitation:** [<u>https://tailscale.com/docs/features/tailscale-ssh</u>](https://tailscale.com/docs/features/tailscale-ssh)

**\[T2\] Tailscale pricing:** [<u>https://tailscale.com/pricing</u>](https://tailscale.com/pricing)

**\[T3\] Tailscale macOS variants:** [<u>https://tailscale.com/docs/concepts/macos-variants</u>](https://tailscale.com/docs/concepts/macos-variants)

**\[MAC1\] Amphetamine App Store - Closed-Display Mode / Drive Alive / Power Protect:** [<u>https://apps.apple.com/us/app/amphetamine/id937984704?mt=12</u>](https://apps.apple.com/us/app/amphetamine/id937984704?mt=12)

## Research interpretation notes

- Open GitHub issues are evidence of reported behavior, not proof that every device/build is affected. The manual therefore converts them into acceptance-test requirements rather than universal claims.

- Commercial prices are point-in-time observations and are explicitly labeled for re-verification before purchase.

- OSS maturity judgments weigh install path, license, current documentation, breadth of CLI support, and evidence of active development; star counts are not treated as a reliability guarantee.

- The hook JSON uses current documented event names and inputs. Re-run /hooks or inspect the live hooks reference after CLI upgrades.

# Appendix A. Decision matrix: choose the smallest stack that satisfies the requirement

| **Need**                                                     | **Use**                                                  | **Do not use as substitute**                              |
|--------------------------------------------------------------|----------------------------------------------------------|-----------------------------------------------------------|
| Answer/steer the exact local Claude conversation from iPhone | Claude Remote Control                                    | Raw SSH terminal alone                                    |
| Guaranteed independent audible attention signal              | Claude hooks -\> Bark or Pushover                        | Native RC push alone until verified                       |
| Recover from invisible/buggy mobile dialogs                  | Tailscale + SSH/Mosh -\> tmux                            | A second push service without shell access                |
| One mobile cockpit for Claude + Codex + other CLIs           | Moshi; or Happy/CC Pocket depending desired architecture | Claude RC, which is Claude-specific                       |
| Durable multi-agent host visibility                          | Herdr after baseline tmux works                          | Installing multiple wrappers before persistence is proven |
| Privacy-sensitive alternative control plane                  | CC Pocket / self-hosted or evaluated Happy architecture  | Assuming first-party RC transcript stays only local       |
| Long-session continuity                                      | Tracker + CONTEXT/ADR/HANDOFF + deliberate compaction    | One immortal chat transcript                              |
| Reduce permission fatigue safely on local Mac                | Sandbox + auto/targeted rules                            | bypassPermissions on the raw laptop                       |
| Parallel edits                                               | Git worktrees / RC --spawn worktree                      | Several agents editing one working directory              |

# Appendix B. Final recommended deployment for the supplied environment

For the supplied MacBook Pro M4 + iPhone 15 Pro + AT&T + Spectrum environment, the recommended deployment is intentionally conservative:

60. Keep the critical current Claude Code conversation local on the Mac and attach first-party Remote Control from inside the existing session.

61. Enable both Claude mobile push toggles, but treat native push as visual/best-effort until it passes the locked-phone AT&T test.

62. Install an independent Bark pager hook now; choose Pushover instead if repeated acknowledgement-style paging is worth \$4.99.

63. At the next safe boundary, run Claude inside tmux. Add Herdr only if you need its blocked/working/done/idle overview for several concurrent agent panes.

64. Install Tailscale on Mac and iPhone, enable macOS Remote Login, use SSH keys, and keep router port-forwarding off.

65. If a better iOS shell/agent inbox is desirable, install Moshi and start with the free tier. It is the best complement for multi-vendor CLI operation.

66. Do not migrate the irreplaceable session into Happy/CC Pocket mid-flight. Trial them on disposable repos; adopt one for future sessions only if its control-plane trade-offs outperform first-party RC for your workflow.

67. Replace --dangerously-skip-permissions on the raw Mac with sandbox + targeted permissions as soon as practical; preserve dangerous bypass only in genuinely isolated environments.

68. Make HANDOFF.md + issue/wayfinder state mandatory before compaction, end of day, or any control-plane migration.

69. Retest the end-to-end doorbell and recovery shell after every material Claude Code / iOS / hook upgrade.

| **Definition of success** You can leave the Mac for an hour, receive an audible iPhone page at the first real HITL gate, answer the same live session from the phone, and - if the mobile UI fails - reach the exact tmux pane over Tailscale and recover without losing context or exposing public SSH. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
