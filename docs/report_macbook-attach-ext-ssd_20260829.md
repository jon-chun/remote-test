# External SSD Connect/Disconnect Automation — System Survey & Fix Plan

**Date:** 2026-08-29
**Scope:** MacBook Pro M4, 24GB unified memory, 1TB internal SSD, sometimes-attached OWC Express 1M2 (4TB, USB4/Thunderbolt, 40Gb/s)

---

## 1. Hardware & current mount state

| Item | Value |
|---|---|
| Host | MacBook Pro M4 (Mac16,1), 10-core (4P+6E), 24GB unified memory, macOS 15.7.5 (24G624) |
| Internal SSD | 1TB (`disk0`/`disk3`), Data volume ~994GB, currently ~370GB free |
| External SSD | OWC Express 1M2 80G, USB4, 40Gb/s link — volume `RESEARCH_SSD` (664GB used of 4TB container) + a Time Machine volume on the same container |
| Mount point | `/Volumes/RESEARCH_SSD` |
| `disksleep` | 10 min — external SSD could theoretically spin/link down after 10 min idle if the enclosure honors macOS disk-sleep signaling; worth ruling out as a cause of "spurious" disconnects |

The drive currently hosts: Ollama model weights (187GB), Hugging Face caches (17GB), an empty `conda_envs` staging dir, LM Studio downloads, an OrbStack VM disk image (25GB), and a Docker Desktop VM disk image (89GB) — i.e., essentially all large local AI/dev tooling state lives off the internal SSD.

---

## 2. Existing custom automation (already built, partially broken)

Three LaunchAgents already implement a hand-rolled version of exactly what this task asks for:

- **`com.user.research-ssd-hotswap`** (`~/Library/LaunchAgents/…plist` → `~/scripts/research-ssd-hotswap.sh`) — uses launchd `WatchPaths` on `/Volumes`, detects `RESEARCH_SSD` presence, flips `launchctl setenv` for `OLLAMA_MODELS`, `HF_HOME`, `HF_DATASETS_CACHE`, `CONDA_ENVS_PATH` between external and internal fallback paths, attempts to restart Ollama, and posts a macOS notification. State is tracked in `~/.cache/research-ssd-state` to avoid redundant restarts.
- **`com.user.ollama-env`** — a simpler `RunAtLoad`-only variant of the same env-var logic, for login-time setup.
- **`com.user.disk-health-check`** (every 6h) — reports free space on both volumes and Ollama model count, with low-space notifications.

This is a reasonable architecture, and the intent (documented in `~/scripts/CHEATSHEET.md`) is correct: auto-detect, remap env vars, restart Ollama, and fall back to an internal `llama3.2:3b` when the drive is absent. **In practice it does not work.**

### 2.1 Confirmed live bug: Ollama never actually restarts

Right now, with `RESEARCH_SSD` mounted and 187GB of real models present, `ollama list` returns **empty**. Root cause, fully traced:

1. `ollama serve` runs as a Homebrew service (`homebrew.mxcl.ollama`, PID has been alive since 2026-08-23 — 6 days, across at least 4 logged mount/unmount cycles in that window).
2. `launchctl setenv OLLAMA_MODELS …` only affects **newly spawned** processes — the GUI-session env — never the already-running `ollama serve` process. This is a known upstream limitation, not a local misconfiguration ([ollama/ollama#4749](https://github.com/ollama/ollama/issues/4749), similarly #7331, #1501).
3. The hotswap script's restart logic is supposed to compensate by force-restarting Ollama on every state change, via a 3-way branch:
   ```bash
   if brew services list 2>/dev/null | grep -q "ollama.*started"; then
       brew services restart ollama …
   elif pgrep -x "Ollama" >/dev/null 2>&1; then
       # kill/relaunch the .app
   else
       echo "Ollama not running; env set for next launch"
   fi
   ```
4. **Both branches fail, every time**, for two independent reasons:
   - Branch 1 (`brew services list`) fails silently under launchd's execution environment: a LaunchAgent with no `EnvironmentVariables.PATH` key gets `PATH=/usr/bin:/bin:/usr/sbin:/sbin` (confirmed via `launchctl print`), which does not include `/opt/homebrew/bin`. `brew` is not found, the command errors, `grep -q` sees no matching output, and the branch is skipped.
   - Branch 2 (`pgrep -x "Ollama"`) checks for the **macOS app's** process name (capital-O `Ollama`), but this installation runs Ollama as a **Homebrew service**, whose process is literally `ollama serve` (lowercase, different argv0) — this check can structurally never match here, regardless of PATH.
   - Every single line in `~/logs/research-ssd-hotswap.log` since the script's creation reads `"Ollama not running; env set for next launch"` — including at timestamps where PID 5410 was demonstrably running. The script has never once successfully restarted Ollama.

Net effect: `launchctl setenv` is updated correctly on every mount/unmount, but the live `ollama serve` process never picks up the new `OLLAMA_MODELS`, so `ollama list` is wrong until the user *manually* runs `brew services restart ollama` (as documented, resignedly, in the CHEATSHEET's own troubleshooting section).

### 2.2 Undocumented risk: OrbStack and Docker Desktop VM disks live on the removable drive

`~/OrbStack` is a symlink to `/Volumes/RESEARCH_SSD/orbstack/data` (containing `data.img.raw`, 25GB), and Docker Desktop's `settings-store.json` sets `"DataFolder": "/Volumes/RESEARCH_SSD/docker_data/DockerDesktop"` (89GB `Docker.raw`). OrbStack was confirmed **running** during this survey with its VM disk on the external drive.

This contradicts the repo's own `~/scripts/CHEATSHEET.md`, which states "Docker data stays on the internal SSD — works with or without the external drive." That claim is stale/incorrect for the current configuration. There is currently **no automation at all** — not even a manual documented procedure — for quiescing OrbStack or Docker Desktop before a disconnect, or for detecting/recovering from a surprise disconnect while their VMs are live. A bus-powered Thunderbolt/USB4 unplug while a VM disk image is open is a real corruption risk, not just a service-availability inconvenience.

### 2.3 Undocumented gap: LM Studio has no fallback

`~/.lmstudio/settings.json` hardcodes `downloadsFolder` to `/Volumes/RESEARCH_SSD/ai_caches/lmstudio`. The CHEATSHEET explicitly notes: "No automated fallback... manually change the models path." Confirmed still true — no script touches LM Studio's config.

---

## 3. Current best practices, researched 2026-08-29

### 3.1 Mount/unmount detection mechanisms

| Mechanism | Verdict |
|---|---|
| launchd `WatchPaths` (current approach) | Apple's own docs call filesystem-event `WatchPaths`/`QueueDirectories` monitoring "highly race-prone" — events can be missed, and it does not fire for volumes already mounted at login. Confirmed by community reports of intermittent misses. Must be paired with idempotent state-check logic (which the existing script already does via its state file — that part is sound). |
| `StartOnMount` | A real launchd key, fires on **any** volume mount system-wide — needs the same self-filtering `WatchPaths` already does. Not obviously better than the current approach. |
| DiskArbitration framework (`DARegisterDiskAppearedCallback`) | The only race-free, event-driven API — but requires a small compiled helper (Swift/ObjC, or Rust via `objc2-disk-arbitration`); no CLI hook. `diskutil activity` streams the same events at the shell level but Apple explicitly warns its output must never be parsed programmatically — fine for manual debugging only. |
| `fswatch` | FSEvents-based, lighter than polling `/Volumes`, but has a known gap: no event fires when a filesystem is mounted **on top of** a watched path until it's unmounted again — not reliable here. Not currently installed. |
| **macOS Tahoe (26) Shortcuts.app native trigger** | **New and significant**: Shortcuts now ships a first-class "External Drive Connected / Disconnected" automation trigger, filterable to a specific drive, that can run a shell-script action directly — no compiled helper, no `WatchPaths` race conditions. This is the most relevant 2025-2026 development for this exact problem. |

**Recommendation:** keep `WatchPaths` as the low-effort trigger (it's "good enough" once the downstream restart logic is fixed), but evaluate migrating the trigger itself to the new Shortcuts external-drive automation for a cleaner, race-free signal — Shortcuts can shell out to the same scripts already written.

### 3.2 Ollama runtime relocation

- **No hot-reload / SIGHUP mechanism exists for `OLLAMA_MODELS`.** A full process restart is the only supported path — confirmed by Ollama's own issue tracker, not a gap in this setup.
- The local bug (setenv doesn't propagate to a running process) is a **known, currently-open upstream issue**: [ollama/ollama#4749](https://github.com/ollama/ollama/issues/4749) is the canonical report of exactly this symptom; [#7331](https://github.com/ollama/ollama/issues/7331) is the same pattern for `OLLAMA_HOST`; [#13404](https://github.com/ollama/ollama/issues/13404) confirms there is **no graceful fallback** upstream when the configured models directory disappears — it fails silently rather than falling back, which matches what's been observed here.
- Correct restart primitives for a Homebrew-managed service (in order of preference): `brew services restart ollama`, or `launchctl kickstart -k gui/$(id -u)/homebrew.mxcl.ollama` (works without requiring `brew` on `PATH` — this sidesteps the root cause of bug 2.1 directly, since `launchctl` is always on the launchd-default `PATH`).
- Homebrew's default service template has no first-class way to inject env vars into the service; the existing `com.user.ollama-env` companion-plist pattern already in use here is the documented community workaround.

### 3.3 OrbStack / Docker Desktop external-disk safety

- **OrbStack**: relocating the data path via Settings → Storage does a clean *migration*, but if the configured external volume is simply **missing at launch**, OrbStack throws a fatal, unrecoverable error dialog with only "Report" or "Quit" — no automatic fallback or in-app reset ([orbstack/orbstack#1726](https://github.com/orbstack/orbstack/issues/1726)). This means OrbStack must be fully stopped *before* the drive disappears, or prevented from auto-launching when the drive is absent — there is no graceful in-app degradation to rely on.
- `orb stop` exists for graceful VM shutdown but has reported reliability gaps ([orbstack/orbstack#745](https://github.com/orbstack/orbstack/issues/745)) — treat it as best-effort, verify the process actually exits.
- **Docker Desktop**: external/relocated disk-image storage is widely reported as broken/hanging in the Docker for Mac issue tracker (#5550, #6803, #6797) — this is a known-fragile configuration upstream, independent of anything specific to this machine.
- **Pre-disconnect ("about to unplug") detection is not physically possible** for bus-powered Thunderbolt/USB4 — removal is an immediate hardware signal; DiskArbitration's "disappeared" callback fires only *after* the fact. The only real prevention mechanism is a **manual pre-eject action** (e.g., a Shortcuts "Eject RESEARCH_SSD" shortcut that runs `orb stop` + `osascript -e 'quit app "Docker"'` first, *then* ejects) — not an automatic safeguard against a surprise physical unplug.

### 3.4 Graceful-degradation architecture — no existing prior art

No dedicated write-up exists for this exact scenario (local AI/dev tooling with model weights and VM disks on a sometimes-attached external SSD on Apple Silicon). Generic "move Ollama to an external drive" guides exist but none address the absent-volume case gracefully — confirmed behavior upstream is silent failure via broken symlink, not fallback. This means the fix implemented here would be closer to a reference implementation than an application of established patterns.

---

## 4. Summary of gaps to close

1. **Fix the Ollama restart branch** in `research-ssd-hotswap.sh` — replace the broken `brew services list` / `pgrep -x "Ollama"` branches with `launchctl kickstart -k gui/$(id -u)/homebrew.mxcl.ollama` (PATH-independent, works regardless of app-vs-brew install), and verify post-restart with a short poll loop calling `ollama list` rather than trusting the restart succeeded blindly.
2. **Add OrbStack/Docker quiesce-on-disconnect handling** — on transition to `unmounted`, if OrbStack or Docker Desktop are running with their data dir now missing, they need to be stopped cleanly (or at minimum the user needs an unmissable, high-urgency alert, since mid-flight VM disk corruption is possible) rather than left running against a vanished disk.
3. **Add a manual "safe eject" Shortcuts/script** that quiesces OrbStack + Docker + Ollama *before* physical removal, since no automatic pre-disconnect signal exists.
4. **Extend coverage to LM Studio**, which currently has zero automation despite being in the same dependency class as Ollama.
5. **Correct the stale CHEATSHEET.md claim** that Docker data stays internal.
6. **Consider migrating the trigger mechanism** from `WatchPaths` to macOS Tahoe's native Shortcuts external-drive-connected/disconnected automation, now that it exists, for a race-free signal — while keeping the existing idempotent state-file check either way.

---

## 5. Sources

- [launchd.plist man page — WatchPaths/QueueDirectories race warning](https://leancrew.com/all-this/man/man5/launchd.plist.html)
- [Apple Developer Forums — WatchPaths reliability](https://developer.apple.com/forums/thread/788105)
- [macmost.com — macOS Tahoe Shortcuts automation, external drive trigger](https://macmost.com/an-introduction-to-shortcuts-automation-in-macos-tahoe.html)
- [Apple Support — Shortcuts automations](https://support.apple.com/en-us/125148)
- [disk-arbitrator (DiskArbitration reference implementation)](https://github.com/aburgh/Disk-Arbitrator)
- [ollama/ollama#4749 — launchctl setenv doesn't propagate to running process](https://github.com/ollama/ollama/issues/4749)
- [ollama/ollama#7331 — same pattern for OLLAMA_HOST](https://github.com/ollama/ollama/issues/7331)
- [ollama/ollama#13404 — no graceful fallback when models dir missing](https://github.com/ollama/ollama/issues/13404)
- [Homebrew service env-var wrapper workaround](https://falko.zurell.de/2025/02/26/adding-environment-variables-to-services-in-home-brew/)
- [orbstack/orbstack#1726 — fatal error when configured external volume missing](https://github.com/orbstack/orbstack/issues/1726)
- [orbstack/orbstack#745 — orb stop reliability](https://github.com/orbstack/orbstack/issues/745)
- [orbstack/orbstack#2354 / #239 — data relocation does not migrate existing data](https://github.com/orbstack/orbstack/issues/2354)
- [docker/for-mac#5550 — external disk image location broken](https://github.com/docker/for-mac/issues/5550)
