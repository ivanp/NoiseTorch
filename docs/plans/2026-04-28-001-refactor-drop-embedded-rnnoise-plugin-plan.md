---
title: "refactor: Drop embedded RNNoise plugin, depend on system noise-suppression-for-voice"
type: refactor
status: completed
date: 2026-04-28
deepened: 2026-04-29
---

# refactor: Drop embedded RNNoise plugin, depend on system noise-suppression-for-voice

> **Note:** This plan was rewritten on 2026-04-29 after `document-review` surfaced fabricated codebase claims in the prior version (non-existent functions, wrong UI stack, wrong line numbers, wrong file paths). The corrected plan reflects the actual `feature/qt6-tray` branch state.

## Overview

Stop bundling the custom `nt-filter` LADSPA plugin (`c/ladspa/rnnoise_ladspa.so`) inside the Go binary via `//go:embed`. Replace with a runtime lookup of werman's `noise-suppression-for-voice` LADSPA plugin (`librnnoise_ladspa.so`) installed via the system package manager.

**Primary motivation:** reduce the fork's maintenance burden. NoiseTorch-ng is a fork of an unmaintained upstream; keeping the embedded C plugin requires tracking rnnoise + c-ringbuf submodules, maintaining the C build, and shipping ~144 KB of binary blob. Dropping it lets the fork shrink to a thin controller around the de facto standard Linux rnnoise LADSPA.

**Target environment: Arch Linux + PipeWire.** `noise-suppression-for-voice` is in Arch `extra` and is the primary tested configuration. Other distros (Debian, Ubuntu, Fedora, etc.) are expected to work since werman is widely packaged, but multi-distro UX polish is explicitly out of scope. AppImage / non-package-manager portability is dropped — users on systems without the package install it themselves or use a different release channel.

Bundled secondary change: gate the existing `capabilitiesView` UI push on `servertype_pulse` so PipeWire-only users stop seeing the CAP_SYS_RESOURCE prompt.

This drops the `os.CreateTemp` /tmp dump-and-delete cycle, deletes the embedded C source tree (~150 KB build artifacts plus rnnoise/c-ringbuf submodules), trims build steps, and aligns NoiseTorch with the de facto standard rnnoise LADSPA on Linux.

## Problem Frame

The current architecture (verified against `feature/qt6-tray` branch as of 2026-04-29):

- `main.go:32-33` embeds `c/ladspa/rnnoise_ladspa.so` as `libRNNoise []byte`.
- `main.go:108-116` `dumpLib()` writes the embedded bytes via `os.CreateTemp("", "librnnoise-*.so")` (i.e. `/tmp/librnnoise-NNNN.so`) on every startup.
- `main.go:118-124` `removeLib()` deletes the temp file on exit (deferred from `main()`).
- `module.go:213,227` (PipeWire load paths) pass `filepath.Base(ctx.librnnoise)` — i.e. just `librnnoise-NNNN.so` — meaning PipeWire/PulseAudio resolve via LADSPA_PATH. This works today because the noisetorch process inherits whatever `LADSPA_PATH` the audio daemon uses, OR PA dlopens the basename via its built-in plugin search. **This indirection is fragile and not documented in the codebase.**
- `module.go:246,290` (PA load paths) pass the full `/tmp/...` path — different from PipeWire path.
- `cli.go:178-181` `cleanupExit(librnnoise, exitCode)` currently calls `removeLib`. 15 call sites across `cli.go`.

The custom plugin (`c/ladspa/module.c`) exposes label `nt-filter` with one control input (VAD threshold). It is not packaged anywhere; it exists only inside the NoiseTorch binary.

The user wants single-binary portability traded for cleaner architecture: rely on the system package `noise-suppression-for-voice` (Arch `extra`, Debian/Ubuntu, Fedora) and let NoiseTorch become a pure controller. Werman's plugin exposes labels `noise_suppressor_mono` and `noise_suppressor_stereo`.

**Note on portability:** the value of the refactor is smaller than first framed — there is no `LADSPA_PATH` config write, no `~/.config/environment.d/noisetorch-ladspa.conf`, no `pipewire-pulse` restart in current code. The actual savings: ~144 KB embedded blob, /tmp write/delete, ~150 KB c/ build artifacts, two submodules, one Makefile target.

## Requirements Trace

- R1. Eliminate the `os.CreateTemp` /tmp dump and corresponding cleanup.
- R2. Eliminate the embedded `libRNNoise` byte slice.
- R3. Eliminate the embedded C source tree and submodules (`c/ladspa/`, `c/rnnoise/`, `c/c-ringbuf/`).
- R4. Use system-installed werman plugin with labels `noise_suppressor_mono` (mic input) and `noise_suppressor_stereo` (output).
- R5. Behavior on missing plugin: hard-fail with stderr install hint **only for paths that need the plugin** (load/unload). Inspection paths (`-l`, `-c`, `-setcap`) must continue to work.
- R6. Preserve existing PulseAudio + PipeWire compatibility paths and module-load behavior.
- R7. Preserve existing user-facing CLI flags and Nucular UI.
- R8. PipeWire-only users never see the CAP_SYS_RESOURCE capabilitiesView. Capability remains a hard runtime requirement for PulseAudio path only.

## Scope Boundaries

- **Out of scope:** Full removal of CAP_SYS_RESOURCE / `pkexec setcap` flow. Capability is still required for PulseAudio rlimit handling (`module.go:loadSupressor`); removing it would break PA-only users.
- **Out of scope:** Switching to PipeWire native `filter-chain.conf` config (alternative considered, deferred — bigger architectural change).
- **Out of scope:** Distro detection or auto-install of the package.
- **Out of scope:** Backwards-compatibility with stale `~/.ladspa/noisetorch_rnnoise.so` from very old NoiseTorch versions. Old file (if present) is orphaned but harmless.
- **Out of scope:** Hybrid embed-fallback architecture (preserve `//go:embed` as fallback when system werman absent). Considered and rejected — keeps c/ tree maintenance burden permanently, defeats primary motivation.
- **Out of scope:** Multi-distro install hints / package-manager auto-detection in errorView. Arch is primary target; other distros get a generic "install noise-suppression-for-voice via your package manager" message.
- **Out of scope:** Single-binary portability for AppImage / non-package-manager users. Acknowledged regression. Users on systems without werman install via their package manager or build it manually.

## Context & Research

### Relevant Code and Patterns (verified line numbers)

- `main.go:24` — `_ "embed"` import.
- `main.go:32-33` — `//go:embed c/ladspa/rnnoise_ladspa.so` + `var libRNNoise []byte`.
- `main.go:69-70` — `rnnoisefile := dumpLib()` + `defer removeLib(rnnoisefile)`.
- `main.go:74` — `ctx.librnnoise = rnnoisefile`.
- `main.go:76` — `doCLI(opt, ctx.config, ctx.librnnoise)`.
- `main.go:82-83` — `ctx.haveCapabilities` and `ctx.capsMismatch` set pre-UI.
- `main.go:85` — `resetUI(&ctx)` called pre-Nucular-window.
- `main.go:94` — `go paConnectionWatchdog(&ctx)` started after window built.
- `main.go:108-116` — `dumpLib()` body: `os.CreateTemp("", "librnnoise-*.so")` → `f.Write(libRNNoise)`.
- `main.go:118-124` — `removeLib()` body: `os.Remove(file)`.
- `main.go:219-254` — `paConnectionWatchdog` reconnection loop, calls `serverInfo()` at line 235 and `resetUI(ctx)` at line 249.
- `cli.go:178-181` — `cleanupExit(librnnoise string, exitCode int)` body: `removeLib(librnnoise); os.Exit(exitCode)`.
- `cli.go` cleanupExit call sites: 15 calls (lines 58, 64, 66, 72, 102, 118, 120, 130, 140, 142, 146, 156, 166, 168, 172).
- `module.go:213,227` — PipeWire load: `label=nt-filter plugin=%s control=%d` with `filepath.Base(ctx.librnnoise)`.
- `module.go:246` — PulseAudio input load: `label=nt-filter plugin=%s control=%d` with full `ctx.librnnoise`.
- `module.go:289-290` — PulseAudio output ladspa-sink: `label=nt-filter ... channels=1 plugin=%s control=%d rate=%d` with full `ctx.librnnoise`.
- `module.go:296` — PA output post-ladspa loopback: `channels=2` (real-sink delivery, already stereo).
- `module.go:302` — PA output pre-ladspa loopback: `channels=1` (feeds mono ladspa-sink today).
- `ui.go:404-...` — `capabilitiesView(ctx, w)` definition.
- `ui.go:491-492` — `if !ctx.haveCapabilities { ctx.views.Push(capabilitiesView) }` inside `resetUI`.
- `c/ladspa/module.c:167` — Plugin label `"nt-filter"`.
- `c/rnnoise/`, `c/c-ringbuf/` — git submodules per `.gitmodules`.
- `Makefile:8,12,33-35` — `dev`/`release` depend on `rnnoise` target which runs `git submodule update --init --recursive` and `$(MAKE) -C c/ladspa`.

### GitNexus Impact Analysis

`dumpLib`: 1 caller (`main`), risk LOW. `libRNNoise`: 0 callers (used only by embed directive), risk LOW.

### External References

Werman plugin: https://github.com/werman/noise-suppression-for-voice

Standard LADSPA install paths (probe order):
- `/usr/lib/ladspa/` (Arch, Fedora)
- `/usr/lib64/ladspa/` (some 64-bit distros)
- `/usr/lib/x86_64-linux-gnu/ladspa/` (Debian/Ubuntu)
- `/usr/local/lib/ladspa/` (manual install)
- Plus `$LADSPA_PATH` if set

Werman mono variant port layout (label `noise_suppressor_mono`, must be confirmed via `analyseplugin` during pre-Unit-3 spike):
- 0: Input (audio)
- 1: Output (audio)
- 2: VAD Threshold (%) (control input)
- 3: VAD Grace Period ms (control input, default 200)
- 4: Retroactive VAD Grace ms (control input, default 0)

Stereo variant (`noise_suppressor_stereo`) presumed: 2 audio in, 2 audio out, same control structure. **Verify before Unit 3.**

PulseAudio's `module-ladspa-sink control=<list>` accepts a comma-separated list and fills control input ports in declaration order. Passing single `control=N` fills port 2 only; ports 3,4 use defaults — verify this PA behavior (single value with multi-control plugin) in pre-spike.

## Key Technical Decisions

- **Lookup is needed for load/unload paths only, not for every CLI flag.** `findSystemPlugin` runs lazily — called from `loadSupressor` (and `unloadSupressor` if needed) rather than at process startup. This means `-c` (update check), `-l` (list devices), `-setcap` continue to work without werman installed. Tray UI also runs without werman; failure surfaces only when user actually toggles a filter.
- **Hard-fail dialog at filter toggle (Nucular `errorView`).** When user toggles a filter and `findSystemPlugin` returns no path, push an `errorView`-style screen with the install hint. Stderr message logged in parallel for CLI users. Avoids the silent-break-on-launch UX failure that would happen with eager process-startup hard-fail.
- **Probe order: standard distro paths first, then `$LADSPA_PATH`.** Reverses the previously-planned order. Prefers distro-managed correctness over user-environment overrides — protects against stale `LADSPA_PATH` set by other audio tools (Ardour, Carla) or legacy noisetorch env.d files. Log every candidate path tried at debug level so users diagnosing wrong-version issues can grep the log.
- **Pass full plugin path to all `module-ladspa-*` loads, including PipeWire.** Drop `filepath.Base()` wrapping at `module.go:213,227`. Both PA and PW accept absolute paths. Eliminates the latent LADSPA_PATH-dependency bug in PipeWire path.
- **Drop `cleanupExit(librnnoise string, exitCode int)` librnnoise param entirely.** With `removeLib` gone, the param is dead weight. New signature: `cleanupExit(exitCode int)`. Touch all 15 call sites.
- **Delete `c/` directory wholesale + remove `.gitmodules` in a separate commit, deferred to a follow-up release.** Wholesale tree deletion is irreversible inside a release line — phase it AFTER werman path is validated in field. Initial release retains `c/` as dead weight; follow-up release deletes it once werman behavior is confirmed across user-reported distros.
- **Mono input label, stereo output label.** Output filter at `loadPulseOutput`/`loadPipeWireOutput` switches to `noise_suppressor_stereo` with `channels=2`. Input keeps mono. Verify stereo plugin port count in pre-spike.
- **Channel-count adjustments (PA output chain):**
  - `module.go:289` ladspa-sink `channels=1` → `channels=2`
  - `module.go:302` pre-ladspa loopback `channels=1` → `channels=2`
  - `module.go:296` post-ladspa loopback already `channels=2` — unchanged
- **Capability dialog gating: gate `ctx.views.Push(capabilitiesView)` at `ui.go:491-492` on `servertype_pulse`.** `resetUI` is called from `main.go:85` (pre-server-detect, with `ctx.serverInfo.servertype == 0`) AND `main.go:249` (post-server-detect inside watchdog). The pre-detect call must NOT push capabilitiesView; the post-detect call pushes only when `servertype_pulse`. Simple gate: `if !ctx.haveCapabilities && ctx.serverInfo.servertype == servertype_pulse`.

## Open Questions

### Resolved During Planning

- **Which plugin?** werman (`noise-suppression-for-voice`).
- **Which labels for input/output?** `noise_suppressor_mono` for mic input, `noise_suppressor_stereo` for output.
- **Behavior on missing plugin?** Lazy lookup at filter-toggle. Push `pluginMissingView` (Nucular) with copy-to-clipboard install command + libnotify notification. Inspection CLI flags work without werman.
- **Probe order?** Distro paths first, `$LADSPA_PATH` last.
- **c/ tree deletion?** Phased — Unit 5 deferred to follow-up release after werman-path validation.
- **Hybrid embed-fallback considered?** Rejected. Permanent c/ tree maintenance burden defeats refactor's primary motivation.
- **Multi-distro UX polish?** Out of scope. Arch is primary target. errorView shows generic "install via your package manager" + Arch-specific command. Other distros' users see same view, install via their package manager.
- **Project positioning?** NoiseTorch-ng repositions as "thin controller around werman's noise-suppression-for-voice". Maintainer-survival rationale stated honestly in Overview.

### Deferred to Implementation (must verify before Unit 3 lands)

- **Werman port layout pre-spike.** Run `analyseplugin /usr/lib/ladspa/librnnoise_ladspa.so` against the actually-installed werman package. Confirm: (a) labels `noise_suppressor_mono` and `noise_suppressor_stereo` exist, (b) port indices match assumption (audio-in, audio-out, VAD-threshold, grace, retroactive), (c) PA `control=N` behavior with single value vs multi-control plugin (does it fill port 2 only, or error?). If layout differs, revise Unit 3.
- **PA module-ladspa-sink absolute-path acceptance.** Test by loading `module-ladspa-sink plugin=/usr/lib/ladspa/librnnoise_ladspa.so` via `pactl`. If PA rejects absolute paths, fall back to passing basename and rely on standard-path resolution.
- **Mono-source / stereo-output asymmetry.** Verify the full PA output chain at `channels=2` works when user's actual sink is mono speakers. PW path same check.
- **Werman grace-period audio behavior vs nt-filter.** A/B record same threshold value, listen for VAD differences. If audibly worse, document in release notes that users may need to re-tune Threshold.

## Phased Delivery

Three phases. Each phase ships independently — Phase 2 may begin only after Phase 1 lands; Phase 3 only after Phase 2 soaks for ~1 week of personal use.

### Phase 1 — Capability dialog gating (independent, low risk, immediate benefit)

Ship the `capabilitiesView` gate first. Standalone change to `ui.go`. Unblocks daily PipeWire usability without touching plugin paths.

- **Includes:** Unit 7.
- **Commit:** `fix: skip CAP_SYS_RESOURCE prompt on PipeWire`.
- **Exit criteria:** PipeWire startup with caps absent → no prompt. PA startup with caps absent → prompt still appears.

### Phase 2 — Werman cutover (the real refactor)

After Phase 1 lands, execute the plugin-source switch. Sequential, file-atomic.

- **Includes:** Unit 0 (spike) → Units 1, 2, 3, 4 → Unit 6 (verify) → Unit 8 commits.
- **Commits (suggested):**
  1. `refactor: lazy lookup of system librnnoise_ladspa.so` (Units 1-3 plus the new `makePluginMissingView`).
  2. `chore: drop rnnoise Makefile target` (Unit 4).
- **Exit criteria:** All Unit 6 smoke tests pass on Arch + PipeWire. ErrorView surfaces correctly when werman absent.

### Phase 3 — Tree cleanup (after Phase 2 soaks)

After ~1 week of personal use confirms werman path works, delete the embedded source tree.

- **Includes:** Unit 5.
- **Commit:** `chore: remove embedded rnnoise C source tree`.
- **Exit criteria:** `git status` clean. `make dev` builds. No reference to `c/` in tree or `.gitmodules`.
- **Tag pre-deletion:** `last-embedded-rnnoise` for clean revert path.

## Implementation Units

- [x] **Unit 0: Pre-implementation spike (verify werman plugin + PA absolute-path semantics)** *(Phase 2)*

**Goal:** Resolve the four pre-Unit-3 deferred questions before any code edits.

**Requirements:** R4

**Dependencies:** None. Werman package installed locally.

**Files:** No code changes.

**Approach:**
- Install werman: `pacman -S noise-suppression-for-voice`. Confirm `/usr/lib/ladspa/librnnoise_ladspa.so` exists.
- Run `analyseplugin /usr/lib/ladspa/librnnoise_ladspa.so`. Capture output: labels, port count per label, port names, control input order.
- `pactl load-module module-ladspa-sink sink_name=test_abs plugin=/usr/lib/ladspa/librnnoise_ladspa.so label=noise_suppressor_mono control=50`. If success, absolute paths work. `pactl unload-module` after.
- A/B record: 30s voice + ambient noise through current binary (nt-filter), then through a manually-loaded werman chain at `control=50`. Listen for grace-period artifacts.
- Record findings in plan as Resolved or as new constraints.

**Test scenarios:** None — verification spike.

**Verification:** Plan updated with confirmed port layout, confirmed absolute-path support, A/B comparison summary.

---

- [x] **Unit 1: Add `findSystemPlugin` + lazy lookup, drop embed and dumpLib in main.go** *(Phase 2)*

**Goal:** Drop `//go:embed`, `libRNNoise`, `dumpLib`, `removeLib`. Add `findSystemPlugin() (string, error)` that probes standard paths and returns the absolute path to `librnnoise_ladspa.so`. Lookup is lazy — called from module-load paths, not at startup.

**Requirements:** R1, R2, R5

**Dependencies:** Unit 0.

**Files:**
- Modify: `main.go`
- Possibly modify: `module.go` (to call findSystemPlugin lazily instead of using ctx.librnnoise) — verify scope at execution.

**Approach:**
- Remove the `_ "embed"` import (line 24) and the `//go:embed` directive + `var libRNNoise []byte` (lines 32-33).
- Delete `dumpLib` (lines 108-116) and `removeLib` (lines 118-124).
- Add `findSystemPlugin() (string, error)` returning either the absolute path or an error with install-hint message:
  - Probe standard paths first (`/usr/lib/ladspa`, `/usr/lib64/ladspa`, `/usr/lib/x86_64-linux-gnu/ladspa`, `/usr/local/lib/ladspa`), then split `$LADSPA_PATH` (colon-separated, skip empty elements).
  - For each dir, `os.Stat(filepath.Join(dir, "librnnoise_ladspa.so"))`. Log every candidate. Return on first hit.
  - On miss, return error with message: `noise-suppression-for-voice not found. Install via your package manager (Arch: pacman -S noise-suppression-for-voice).` Single primary command (Arch); generic fallback hint for other distros.
- Update `main.go:69-70` flow: drop `dumpLib`/`removeLib`. Set `ctx.librnnoise = ""` initially (sentinel for "not yet looked up"). Lookup happens lazily inside module loaders.
- Drop the import of `_ "embed"`.

**Patterns to follow:** `log.Printf` style for diagnostic output; existing `log.Fatalf` is NOT used here — failures bubble back to caller, not process-fatal.

**Test scenarios:**
- Happy path: plugin at `/usr/lib/ladspa/`, `findSystemPlugin()` returns absolute path, no error.
- Edge case: `$LADSPA_PATH=/foo:/bar`, plugin only in `/usr/lib/ladspa`, lookup still finds it (standard paths probed first).
- Edge case: `$LADSPA_PATH` contains empty element (`:/foo:`) — skipped, no panic.
- Error path: plugin in no probed path, returns descriptive error mentioning `noise-suppression-for-voice` and install commands.
- Integration: full `main()` startup with plugin absent succeeds (no fatal at startup), tray opens, errorView appears only when user toggles a filter.

**Verification:** `go build` clean. `bin/noisetorch -log` with werman absent shows tray, no startup fatal. `bin/noisetorch -log` with werman present logs `Found rnnoise plugin: /usr/lib/ladspa/librnnoise_ladspa.so` when first filter toggle attempt happens.

---

- [x] **Unit 2: Drop `librnnoise` param from `cleanupExit` (15 call sites) in cli.go** *(Phase 2)*

**Goal:** With `removeLib` gone, `cleanupExit` no longer needs the path.

**Requirements:** R1

**Dependencies:** Unit 1.

**Files:**
- Modify: `cli.go`

**Approach:**
- Change `func cleanupExit(librnnoise string, exitCode int)` to `func cleanupExit(exitCode int)` at `cli.go:178`.
- Update all 15 call sites in `cli.go` (lines 58, 64, 66, 72, 102, 118, 120, 130, 140, 142, 146, 156, 166, 168, 172) to drop the first arg.
- `doCLI(opt CLIOpts, config *config, librnnoise string)` signature: keep `librnnoise` param for now (still threaded into `ctx.librnnoise`); revisit in Unit 3 once we know whether ctx.librnnoise is still used (lazy lookup may make it dead).

**Patterns to follow:** Existing `cleanupExit` call structure.

**Test scenarios:**
- Happy path: `-l` exits 0 cleanly.
- Happy path: `-c` exits 0 cleanly without werman installed (lazy lookup).
- Error path: `-i` with nonexistent source ID exits 1.
- Integration: `pkexec noisetorch -setcap` succeeds without werman installed.

**Verification:** `go build` clean.

---

- [x] **Unit 3: Switch labels to werman + adjust channel counts in module.go** *(Phase 2)*

**Goal:** Replace `label=nt-filter` with `noise_suppressor_mono` (input) / `noise_suppressor_stereo` (output). Pass full plugin path. Adjust output-chain channel counts. Use lazy `findSystemPlugin` lookup at load time.

**Requirements:** R4, R5, R6

**Dependencies:** Unit 0 (port layout confirmed), Unit 1 (`findSystemPlugin` exists).

**Files:**
- Modify: `module.go`

**Approach:**
- At top of `loadSupressor` (or in each `load*` function), call `findSystemPlugin()`. If error, return the error to caller — caller surfaces via UI (push errorView with message).
- `loadPipeWireInput` (`module.go:208-220`): change `label=nt-filter` → `label=noise_suppressor_mono`. Replace `filepath.Base(ctx.librnnoise)` with the resolved absolute path. Keep `channels=1`.
- `loadPipeWireOutput` (`module.go:222-234`): change to `label=noise_suppressor_stereo`. Use absolute path. Bump `channels=1` → `channels=2`. Verify against pre-spike findings.
- `loadPulseInput` (`module.go:236-275`): change `label=nt-filter` → `label=noise_suppressor_mono` at line 246. Plugin path already absolute. Keep mono routing throughout.
- `loadPulseOutput` (`module.go:277-307`):
  - Line 289 ladspa-sink: change `label=nt-filter` → `label=noise_suppressor_stereo`, `channels=1` → `channels=2`.
  - Line 296 post-ladspa loopback: already `channels=2` — unchanged.
  - Line 302 pre-ladspa loopback: change `channels=1` → `channels=2` to match new stereo ladspa-sink.
- Drop `path/filepath` import from `module.go` if no other use remains (verify).
- Caller of `loadSupressor` (UI toggle handler) must handle the new error path: push the plugin-missing view via `ctx.views.Push(makePluginMissingView(ctx))` when err != nil.
- **Add `makePluginMissingView(ctx *ntcontext) ViewFunc` factory in `ui.go`.** Pattern matches existing factories `makeErrorView` (line 436), `makeFatalErrorView` (line 451), `makeConfirmView` (line 466). Returned view displays:
  - Heading: "Plugin not found".
  - Message: `noise-suppression-for-voice` not installed. Probed paths (formatted list, debug only — keep concise in main view).
  - Primary button "Copy install command" → write `pacman -S noise-suppression-for-voice` to clipboard. Use `(*ctx.masterWindow).ActivateEditMode(...)` / nucular clipboard if available, else fall back to `exec.Command("xclip", "-selection", "clipboard")` piping the command string to stdin. `os/exec` already imported in ui.go (line 11).
  - Secondary button "Retry" → re-call `findSystemPlugin`. On success, pop the view and proceed; on miss, leave view up.
  - Tertiary button "OK" / "Close" → pop view back to mainView.
  - On first push per process (boolean latch on `ntcontext` or package-level `sync.Once`), fire `exec.Command("notify-send", "NoiseTorch", "noise-suppression-for-voice not installed").Run()` best-effort. Ignore errors (notify-send may be absent on minimal installs).

**Patterns to follow:** Existing view-factory pattern at `ui.go:436-485`. Button handling pattern: `if w.ButtonText("..."){ ... }` (e.g., `ui.go:115, 226, 252, 378, 394, 417`). View-push: `ctx.views.Push(...)`. View-pop: existing examples in `makeErrorView` body. `exec.Command` already used in `capabilitiesView`-bound `pkexecSetcapSelf` flow.

**Test scenarios:**
- Happy path PipeWire input: enable filter, "Filtered Microphone" source appears.
- Happy path PipeWire output: stereo headphone path produces denoised output without channel-count errors.
- Happy path Pulse input: verify all 4 modules load (null-sink, ladspa-sink, loopback, remap-source).
- Happy path Pulse output: verify all 5 modules load (null-sink x2, ladspa-sink, loopback x2 — both at channels=2).
- Edge case: toggle filter on/off rapidly, no leaked modules.
- Error path: werman absent — toggle attempt surfaces `pluginMissingView` with install hint, no module load attempted, no PA-side error log spam.
- Error path: `pluginMissingView` "Copy install command" button puts `pacman -S noise-suppression-for-voice` on clipboard (verify with `xclip -o -selection clipboard`).
- Error path: `pluginMissingView` first display fires libnotify desktop notification (verify visually). Subsequent displays during same process do not re-fire.
- Error path: `pluginMissingView` "Retry" button after werman install dismisses view and proceeds with toggle.
- Integration: speak into mic, denoising audible via `pavucontrol` listening to "Filtered Microphone".

**Verification:** `go build` clean. Manual tray toggle works on PipeWire. `pactl list modules | grep -E 'nui_|Filtered'` confirms expected module set after each toggle.

---

- [x] **Unit 4: Drop `rnnoise` build target from Makefile** *(Phase 2)*

**Goal:** Remove the C build step from `dev` and `release`.

**Requirements:** R3

**Dependencies:** Unit 1 (no more embed reference). Unit 5 NOT required (Makefile change can ship before tree deletion).

**Files:**
- Modify: `Makefile`

**Approach:**
- Drop `rnnoise` from `dev: rnnoise qt6check` (line 8) → `dev: qt6check`.
- Drop `rnnoise` from `release: rnnoise qt6check` (line 12) → `release: qt6check`.
- Delete the `rnnoise:` target body at lines 33-35.

**Test scenarios:** None — pure build config.

**Verification:** `make dev` runs without `gcc` or `make -C c/ladspa`. Output binary present.

---

- [x] **Unit 5: Delete `c/` tree and `.gitmodules`** *(Phase 3 — after Phase 2 soaks ~1 week)*

**Goal:** Remove orphaned C source, build artifacts, and submodule pointers — but only after werman path is validated in field.

**Requirements:** R3

**Dependencies:** Werman-based release in field for one release cycle with no critical bugs. Tag pre-deletion commit (`last-embedded-rnnoise`) for clean revert.

**Files:**
- Delete: `c/` (entire directory).
- Delete: `.gitmodules`.

**Approach:**
- Tag `last-embedded-rnnoise` on the pre-deletion HEAD before tree removal.
- `git rm -r c/` — removes tracked files.
- `git rm .gitmodules`.
- If submodules were checked out (`git submodule status` shows entries): `git submodule deinit -f c/rnnoise c/c-ringbuf` and `rm -rf .git/modules/c/`. If never initialized, deinit will fail — that's OK, skip it.
- Inspect for untracked .o/.so artifacts in `c/ladspa/` (`ls c/ladspa/*.o c/ladspa/*.so 2>/dev/null`); `rm` any survivors.

**Test scenarios:** None — pure file removal.

**Verification:** `ls c/` returns "No such file or directory". `git status` shows deletions only. `git config -f .gitmodules --list` returns empty.

---

- [x] **Unit 6: Build verification + manual smoke test (initial release)** *(Phase 2)*

**Goal:** Confirm clean compile and end-to-end runtime behavior for the werman-based release (without c/ deletion).

**Requirements:** R1, R2, R4, R5, R6, R7

**Dependencies:** Units 1-4, Unit 7.

**Files:** No changes.

**Approach:**
- Confirm werman: `ls /usr/lib/ladspa/librnnoise_ladspa.so`.
- `make dev` → clean build.
- `bin/noisetorch -log` → tray opens. No startup fatal regardless of werman presence (lazy lookup).
- Toggle input filter → "Filtered Microphone" source appears (`pactl list sources`).
- Toggle output filter → "Filtered Headphones" sink appears.
- Toggle off both → all `nui_`/`Filtered` modules cleaned up (`pactl list modules | grep -E 'nui_|Filtered'` empty).
- Rename plugin (`sudo mv /usr/lib/ladspa/librnnoise_ladspa.so /tmp/.bak`). Restart binary, toggle filter → errorView with install hint, no PA error log noise. Restore plugin (`sudo mv /tmp/.bak /usr/lib/ladspa/librnnoise_ladspa.so`). **Critical:** verify restoration before moving on.
- Drop binary caps (`sudo setcap -r bin/noisetorch`). On a PA system, toggle filter → capabilitiesView appears. On PipeWire, toggle filter → no capabilitiesView, filter loads normally.

**Verification:** All steps succeed.

---

- [x] **Unit 7: Gate capabilitiesView push on `servertype_pulse` in ui.go** *(Phase 1 — ship first)*

**Goal:** PipeWire users no longer see CAP_SYS_RESOURCE prompt. PulseAudio users still see it when caps missing.

**Requirements:** R8

**Dependencies:** None (independent of plugin refactor).

**Files:**
- Modify: `ui.go` (the `resetUI` function around line 491).

**Approach:**
- Locate the `if !ctx.haveCapabilities { ctx.views.Push(capabilitiesView) }` block at `ui.go:491-492`.
- Add server-type guard: `if !ctx.haveCapabilities && ctx.serverInfo.servertype == servertype_pulse { ctx.views.Push(capabilitiesView) }`.
- Reasoning for `resetUI` call sites:
  - `main.go:85` calls `resetUI` BEFORE `paConnectionWatchdog` runs. At this point `ctx.serverInfo.servertype == 0` (zero value). Guard correctly fails — no premature push.
  - `main.go:249` (inside watchdog) calls `resetUI` AFTER `serverInfo()` resolves. At this point servertype is set. Guard fires only on PA.
- No latch needed — `resetUI` clears views and re-pushes deterministically; rapid reconnects naturally re-evaluate the guard.

**Patterns to follow:** Existing `ctx.views.Push(view)` pattern in `resetUI`.

**Test scenarios:**
- Happy path: PipeWire user without caps → capabilitiesView never pushed. Tray usable.
- Happy path: PulseAudio user without caps → capabilitiesView appears after watchdog connects.
- Edge case: User on PipeWire, audio server unreachable initially → resetUI runs at startup with servertype=0, no push (correct — no PA confirmed yet). On reconnect failure, watchdog stays in connectView; capabilitiesView never inappropriately appears.
- Edge case: User toggles between PA and PW (e.g., switches audio backend) — each watchdog reconnect re-runs `resetUI` and re-evaluates. Correct behavior.
- Integration: `pkexec noisetorch -setcap` flow still works end-to-end (this flow exits via `cli.go` before reaching UI).

**Verification:** Manual: launch on PipeWire-only system without caps, no prompt. `setcap -r` on PA system, launch, prompt appears.

---

- [ ] **Unit 8: Pre-commit `gitnexus_detect_changes` + commit** *(runs once per phase)*

**Goal:** Verify scope per `CLAUDE.md`. Stage and commit.

**Requirements:** R1-R8 housekeeping.

**Dependencies:** Units 1-4, 6, 7. (Unit 5 is deferred to a separate follow-up release.)

**Files:** No code changes.

**Approach:**
- Run `gitnexus_detect_changes(scope: "unstaged")`. Review affected processes/risk.
- Suggested commit split:
  1. Plugin lookup refactor (Units 1-3): `refactor: depend on system noise-suppression-for-voice plugin`.
  2. Build cleanup (Unit 4): `chore: drop rnnoise Makefile target`.
  3. Capability dialog gating (Unit 7): `fix: skip CAP_SYS_RESOURCE prompt on PipeWire`.

**Test scenarios:** None.

**Verification:** Detect-changes risk LOW or expected MEDIUM, no surprise affected processes outside `main`/`doCLI`/`loadSupressor`/`unloadSupressor`/`resetUI`.

## System-Wide Impact

- **Interaction graph:** `main()` (no longer calls `dumpLib`) → `doCLI()` / Nucular tray → user toggle → `loadSupressor()` → `findSystemPlugin()` (lazy) → `loadPipeWire*` / `loadPulse*` (label + path changes). `cleanupExit()` simplified, `removeLib` chain dropped.
- **Error propagation:** Plugin missing now surfaces via Nucular `errorView` only at filter-toggle time (was previously a startup `log.Fatalf` path inside `dumpLib` if `os.CreateTemp` failed). Less aggressive failure mode — tray remains usable for users who don't toggle filters.
- **State lifecycle risks:** Old user installs may have stale `~/.ladspa/noisetorch_rnnoise.so` (from very old NoiseTorch versions, predating /tmp dump). Harmless. Optional one-shot cleanup deferred.
- **API surface parity:** No CLI flag changes. Tray Nucular UX unchanged except for new errorView content. Behavior may differ slightly because `nt-filter` vs werman have different VAD characteristics — same RNNoise model, but werman adds grace-period semantics. Document Threshold-retune note in release notes if A/B in Unit 0 reveals audible difference.
- **Integration coverage:** End-to-end audio path smoke-tested manually (Unit 6). No automated coverage exists in repo.
- **Unchanged invariants:** PulseAudio module orchestration, CAP_SYS_RESOURCE rlimit handling for PA, Nucular UI architecture, config schema, update-check flow, `setcap` self-elevation flow.

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| Werman label/port layout differs from assumption | Unit 0 spike verifies via `analyseplugin`. Block Unit 3 on confirmation. |
| PA `module-ladspa-sink` rejects absolute path | Unit 0 spike verifies via `pactl load-module`. Fall back to basename if rejected. |
| Stereo plugin requires more channel-count changes than line 289+302 | Unit 0 spike + Unit 6 smoke test on mono speakers AND stereo headphones. |
| Werman not installed on user's system | Lazy lookup + errorView at toggle time. Tray still usable. Install hint in error message. |
| Werman version drift between distros breaks port mapping | Unit 0 documents tested version. Release notes mention. No runtime probe in v1; consider adding `analyseplugin`-style descriptor inspection in v2. |
| `$LADSPA_PATH` set by other tools shadows system werman | Probe-order reversed — distro paths first. Log all probed paths at debug level. |
| Loss of single-binary portability for AppImage / non-package-manager users | **Accepted.** Out of scope per scope boundaries. AppImage release channel discontinued; users on non-package-manager systems install werman manually or build from source. |
| Submodule deinit fails on uninitialized state | Deletion deferred to follow-up; deinit only attempted if submodules are checked out. |
| Wholesale `c/` tree delete is irreversible inside a release | **Phased rollout: Unit 5 deferred to follow-up release.** Tag `last-embedded-rnnoise` before deletion. |
| Werman grace-period defaults audibly different from nt-filter | Unit 0 A/B test. Release notes mention threshold re-tune if needed. |
| `-c` / `-l` / `-setcap` would break with eager hard-fail | Lazy lookup makes these flags work without werman installed. |

## Documentation / Operational Notes

- Update README post-merge: install requirement, AUR PKGBUILD dependency.
- Update project `CLAUDE.md` if it references the embed/dump architecture.
- Release notes: hard runtime dependency on `noise-suppression-for-voice`; PipeWire users no longer see CAP_SYS_RESOURCE prompt; possible Threshold re-tune if A/B in Unit 0 reveals audible difference.
- AUR PKGBUILD: add `noise-suppression-for-voice` to `depends=()` array.

## Sources & References

- Conversation context (in-session brainstorm).
- Werman plugin: https://github.com/werman/noise-suppression-for-voice
- Arch package: https://archlinux.org/packages/extra/x86_64/noise-suppression-for-voice/
- LADSPA SDK: http://www.ladspa.org/ladspa_sdk/overview.html
- PulseAudio `module-ladspa-sink`: https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Modules/#index35h3
- GitNexus impact analysis (in-session): `dumpLib` upstream LOW (1 caller), `libRNNoise` upstream LOW (0 callers).
- document-review pass 2026-04-29: feasibility-reviewer surfaced ~13 codebase-divergence findings driving this rewrite. coherence, product-lens, adversarial reviews surfaced strategic and decision-deferral findings (presented separately).
