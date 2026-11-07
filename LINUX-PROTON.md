# Running this map on Linux under Proton

This branch makes the Enshrouded Interactive Map work on Linux under Proton/Wine, **including
live player tracking while the game is running**. Upstream builds cannot do that: the window
stays blank, or the app dies silently a minute or two in.

Four commits against `v1.1.1` (`8032bd3`). Two files changed:
`app/src-tauri/tauri.conf.json` and `app/src-tauri/src/tracker.rs`.

Nothing here changes Windows behaviour, apart from turning off the webview's internal
drag-and-drop handler, which this app does not use.

| | Upstream on Linux | This branch |
|---|---|---|
| WebView2 GPU child processes | 12–18, crash-looping | **0** |
| Lifetime with the game running | dies 35 s – 4 min in | **8 min 26 s clean**, then still running |
| Window contents | blank | the map draws |
| Mouse cursor inside the window | missing | works |
| Live player marker | — | follows the character |

Tested on Fedora 44, KDE Plasma 6.7 (Wayland), AMD RX 9070 XT, Proton Experimental, WebView2
runtime `157.0.4341.42`, game build `26430625`.

---

## The three WebView2 problems

None of these are bugs in this app. They are what happens to **any** Tauri 2 + WebView2
application under Wine, and they stack.

### 1. The GPU process crash-loop

WebView2 runs its graphics in a separate process. Under Wine that process dies immediately and
is restarted, over and over:

```
--type=gpu-process --gpu-recent-crash-count=0
--type=gpu-process --gpu-recent-crash-count=1
--type=gpu-process --gpu-recent-crash-count=2
--type=gpu-process --gpu-recent-crash-count=3   -> Chromium stops retrying, nothing paints
```

12–18 attempts per launch. There is no Wine exception or backtrace, because the children are
terminated deliberately (`NtTerminateProcess ... exit_code -36862`, i.e. `0xFFFF7002`) —
Chromium killing its own child, not a segfault.

Root cause: **[WineHQ bug 58921](https://bugs.winehq.org/show_bug.cgi?id=58921)**. Wine does
not implement DirectComposition — `DCompositionCreateDevice` returns `E_NOTIMPL` — and the
D3D11 shared-texture path Chromium composites through is broken.
([Bug 59370](https://bugs.winehq.org/show_bug.cgi?id=59370) is the same thing, closed as a
duplicate.)

**Fix:** `--in-process-gpu`. The GPU threads run inside the browser process, so the broken
cross-process path is never used at all. Confirm it worked by counting: there should be **no**
GPU child process whatsoever.

### 2. The silent killer

Even when it drew, the app died between 35 seconds and 4 minutes in — no dialog, no log entry,
no crashpad dump.

Tauri calls `RegisterDragDrop` in WebView2's browser process, and `RevokeDragDrop` later runs
in the app's own process. Wine then dereferences a pointer belonging to a different process:

```
Unhandled exception: page fault on read access ... (0x006fffff354d3d)    = ole32+0x34d3d
```

This is **[WineHQ bug 60225](https://bugs.winehq.org/show_bug.cgi?id=60225)**, filed against a
different Tauri app, same function.

**Fix:** `"dragDropEnabled": false` on the window. That maps to wry's
`disable_drag_drop_handler`, so `RegisterDragDrop` is never called and the bug cannot fire. No
binary patching of `ole32.dll` needed — which matters, because a patched system DLL would be
silently reverted by the next Proton update.

### 3. No first frame

Wine has no DWM occlusion tracking, so Chromium decides the window is hidden and never presents
anything.

**Fix:** `--disable-features=CalculateNativeWinOcclusion`.

### Why this has to be a source change

Both of the usual ways to pass Chromium flags into WebView2 failed here:

- **`WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS`** is ignored, because Tauri sets
  `additional_browser_args` itself and that overrides the environment variable. Verified by
  grepping the actual child command lines: the flags appear **zero** times.
- **The HKLM policy** `Software\Policies\Microsoft\Edge\WebView2\AdditionalBrowserArguments`
  also never arrived. All three value names were tried — the exe name, the Tauri AUMID and
  `"*"` — and confirmed written to `system.reg`. The flag still did not reach the child
  processes and 18 GPU processes still spawned. Why is unknown; it was not pursued once setting
  the args in the app's own config worked.

So the args go in `tauri.conf.json`, which is why this branch exists rather than a launch script.

### A trap if you edit the args

`additionalBrowserArgs` **replaces** wry's defaults rather than adding to them, so
`msWebOOUI`, `msPdfOOUI` and `msSmartScreenProtection` must be merged back into our own
`--disable-features=` list — they are already in there. Chromium also does not reliably honour
`--disable-features` twice, so keep exactly one of it in the string.
See the [Tauri configuration reference](https://v2.tauri.app/reference/config/).

Changing these args also wants a fresh webview data directory. Move the app's `EBWebView`
folder aside after updating.

---

## The fourth problem: the marker stuck in the corner

With the window finally painting alongside the game, a Linux-specific defect appeared that
nobody had reached before. The app reported **"Character located"** and pinned the marker in
the bottom-left corner of the map, where it stayed however far the character walked.

The reading was not zero. It was:

```
debug ponteiro: x=1.3 y=0.0 z=5.0 mudou=false
```

On a 10,240 m map that **displays** as `X 0.0 · Z 0.0`. Under Wine the fixed-offset pointer path
in `tracker.rs` resolves an object holding that frozen value, and three things made it
permanent:

1. `in_world()` accepted it, so the offsets cache in `offsets()` kept the bad pointer for the
   life of the process — a cached hit skips the 2-second re-search forever.
2. `run_pointer()` loops until the game's PID changes. A reading that *succeeds*, even a wrong
   one, means it never exits, so the heuristic scanner was unreachable in practice.
3. The scanner was a dead end in the other direction: once entered, it was left only when the
   game process changed.

**Reading the game's memory under Wine works fine.** That is the useful finding. The proof is
the scanner itself — forced on with `EMB_HEURISTIC=1`, it found the character within seconds of
walking:

```
debug: com pontos=318 melhor=Some(((1498, 566), 138))
prim=Some(de1b0) q=129
```

318 moving candidates, 138 copies in the winning cell, while the pointer path sat on 1.3 / 5.0.
Only the fixed-offset path fails.

**Fix,** in three parts:

- `in_world()` requires `x > 1` and `z > 1`, the same bar `cam_ok()` already applies to the
  camera. Nothing within a metre of the world origin is a character.
- `run_pointer()` gives up if the position has never moved more than 0.5 m within 15 s of the
  first read, or if there was no valid reading at all in 15 s. Once the position has moved once,
  the check disarms for the life of the process, so standing still later costs nothing.
- The scanner hands control back after 2 min with nothing locked, so neither finder is a dead
  end.

`EMB_DEBUG=1` now also prints the raw pointer-mode reading. Without that, the only symptom was
*an absence of scanner output*, which is consistent with several different causes and is what
made this slow to diagnose.

**What to expect at runtime:** the marker sits in the corner for about 15 seconds, the status
changes to looking for the character, and it locks on **once you move**. The scanner identifies
the player by what moves together, so standing still never locks.

This part is defensive rather than Linux-specific. The offsets are tied to game build numbers;
any user whose offsets go stale after a game patch gets the same frozen marker, and would now
fall back to the scanner instead of being told the character had been found.

---

## Building it

The repo already builds on `windows-latest` in `.github/workflows/build.yml`, with a
`workflow_dispatch` trigger, so **no Windows machine and no local Rust toolchain are needed**:

```bash
gh workflow run build.yml --repo <your-fork> --ref linux-proton-webview2
gh run download <run-id> --repo <your-fork> -n Mapa-Interactive-Enshrouded
```

About 8 minutes. One gotcha: a fresh fork has its workflows unregistered, and there is no API
for that — `actions/permissions` can report `enabled: true` while `gh workflow run` still fails
with `404 workflow build.yml not found on the default branch`. Click once on the fork's Actions
tab: *"I understand my workflows, go ahead and enable them"*.

## Running it

Start the game first, then launch the map into the game's Wine session — it has to share that
session, because it finds the game with `CreateToolhelp32Snapshot`, which only lists processes
in the same session:

```bash
protontricks-launch --appid 1203620 --no-background-wineserver "/path/to/Mapa Interactive Enshrouded.exe"
```

Add `EMB_DEBUG=1` when diagnosing. Before any clean test, check no stale `wineserver` is alive
(`pgrep -xc wineserver` should read `0`) — one outlived the game here for hours and invalidated
an afternoon of A/B testing.

**Fog of war needs a symlink.** The prefix's `SteamPath` registry value points at
`C:\Program Files (x86)\Steam`, a Proton stub with no `userdata`, so the app finds no saves.
Link the prefix's `Steam/userdata` to the real one. Read-only, nothing is copied.

## Things that do not work — don't spend time on them

| Tried | Result |
|---|---|
| `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS` | overridden by Tauri; flag count 0 |
| HKLM `AdditionalBrowserArguments` policy, all three value names | never reached the child processes |
| `protontricks-launch --no-bwrap`, with and without the game running | identical failure |
| Software Vulkan (lavapipe), wined3d + software GL | identical failure |
| GE-Proton 11-1 and 10-34 | the **game** fails to start under GE here, so untestable |

## Verifying a build

Launching proves nothing — upstream launches fine with the game closed. With the game running:

```bash
# 1. the flags actually arrived (must contain in-process-gpu)
tr '\0' ' ' < /proc/$(pgrep -f '[m]sedgewebview2' | head -1)/cmdline

# 2. no GPU child process at all (must be 0)
pgrep -f '[m]sedgewebview2.*type=gpu' | wc -l
```

Then leave it running 5 minutes or more, and walk around to confirm the marker tracks.

Two measurement traps worth knowing, both of which produced confident wrong answers here:
`pgrep` **without** `-f` reports `0` for `msedgewebview2.exe`, because Linux truncates process
names at 15 characters — and `pgrep -f` matches its own shell, so bracket the first letter and
keep the searched-for words out of every other word on the command line, including file paths.

## Credit

The flag set came from **[khoipm06/wuwa-tappo-wine](https://github.com/khoipm06/wuwa-tappo-wine)**,
which runs a different Tauri 2 + WebView2 app under Proton and hit the same GPU crash-loop.
Its `docs/root-cause.md` and `docs/flags.md` are what pointed at `--in-process-gpu`, and its
delivery-channel table independently matches what was measured here: the environment variable
reaches the process environment but never the argv.

Thanks also to the commenter on the mod's Nexus page who reported the identical wall in October
— *"I can't get it to run on Linux. At least not while the game is running"* — along with the
observation that Cheat Engine could read the game's memory perfectly well. Knowing the problem
was not one broken setup, and that memory access was never the issue, saved a great deal of time.

Upstream is [nicolasferreiraxn/Enshrouded-Interactive-Map](https://github.com/nicolasferreiraxn/Enshrouded-Interactive-Map),
MIT licensed. These changes are offered back under the same licence.
