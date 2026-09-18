# macOS arm64 Native Plugin Loading Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make BepInEx load plugins when Valheim runs as a native arm64 process on Apple Silicon, so modded play no longer requires Rosetta.

**Architecture:** The failure is silent, so this plan does not start with a fix. Tasks 1-4 build a reproducible harness and turn on diagnostics that already exist in the source but are compiled out of release builds. Task 5 is a decision gate: the diagnostics name the failing call, and only then is the fix written. Task 6 upstreams the diagnostics defect on its own merit, whatever the outcome.

**Tech Stack:** C (UnityDoorstop), xmake, C# (BepInEx 5.4.23.5), macOS 15 on Apple M4 Pro, Valheim (Unity 6000.0.75f1, Mono).

**Spec:** None. This plan is the output of a debugging session, and the Evidence section below carries what a spec normally would.

## Global Constraints

- Target BepInEx version is exactly **5.4.23.5**. Tag `v5.4.23.5` in this repository.
- Target Doorstop version is exactly **4.5.0**. BepInEx 5 consumes it as a prebuilt release zip; see `build.cake:7`, `DOORSTOP_VER = "4.5.0"`.
- Doorstop source lives in a separate repository, forked to **`dobbo-ca/unitydoorstop`**. Most of the work in this plan is there, not here.
- The game install root, referred to below as `$V`, is
  `/Users/christopherdobbyn/Library/Application Support/Steam/steamapps/common/Valheim`.
- **Never edit `$V/run_bepinex.sh` in place.** A Steam update overwrites it. Work on copies.
- **Never `git merge`.** Rebase branches; merge only through pull requests.
- Do not open a pull request from this fork without setting the target explicitly. `gh pr create` in a fork defaults to the **upstream** repository, `BepInEx/BepInEx`.

---

## Evidence

Measured on 2026-09-18, on the M4 Pro, against the live install.

| Run | Arch | Doorstop injected | BepInEx preloader | Plugin loaded | Log written |
|---|---|---|---|---|---|
| `open -a Valheim.app` | ARM64 | no | no | no | no |
| `run_bepinex.sh` (stock) | x86_64 | yes | yes | yes | yes |
| `run_bepinex.sh` with `ARCHPREFERENCE="arm64"` | ARM64 | **yes** | assemblies mapped | **no** | **no** |

The arm64 row is the bug. `lsof` on that process showed `libdoorstop.dylib`, `BepInEx.dll`, `0Harmony.dll`, `MonoMod.RuntimeDetour.dll` and five more BepInEx core assemblies all mapped into the process, and `Dobbo.Tracker.Client.dll` absent. `BepInEx/LogOutput.log` was not created or appended.

Four explanations were tested and refuted:

1. **Missing arm64 slice.** Refuted. `lipo -archs` reports `x86_64 arm64` for `Valheim.app/Contents/MacOS/Valheim`, `libdoorstop.dylib`, `libmonobdwgc-2.0.dylib` and `libmono-native.dylib`.
2. **Different Mach-O link format per slice.** Refuted. `otool -l -arch x86_64` and `-arch arm64` both report `LC_DYLD_INFO_ONLY` and neither reports `LC_DYLD_CHAINED_FIXUPS`, so `plthook` takes the same code path on both.
3. **LaunchServices or window activation.** Refuted for the plugin failure. The arm64 run registers normally and becomes the front application.
4. **The Dobbo tracker mod.** Refuted. It is architecture-neutral IL, and it is never reached.

The cause of the **silence** is known, and it is the starting point of this plan. In `src/util/logging.h` of UnityDoorstop:

```c
#if VERBOSE
  /* real LOG(), ASSERT(), ASSERT_F() */
#else
#define LOG(message, ...)
#define ASSERT_F(test, message, ...)
#define ASSERT(test, message)
#define ASSERT_SOFT(test, ...)
#endif
```

Release builds do not define `VERBOSE`, so every diagnostic in `src/nix/entrypoint.c` is compiled to nothing — including the four that report a failed hook:

- `"Failed to open current process PLT! Cannot run Doorstop! Error: %s"`
- `"Failed to hook dlsym, ignoring it. Error: %s"`
- `"Failed to hook fclose, ignoring it. Error: %s"`
- `"Failed to hook jit_init_version, ignoring it. ... Error: %s"`

Meanwhile `src/nix/plthook/plthook_osx.c:49-52` hard-defines four debug macros to `1` with no `#ifdef` guard, so the release binary writes about 7,100 lines of bind-opcode dump to stderr on every launch. Both a native run and a Rosetta run produced that dump and neither produced one `[Doorstop]` line.

The build already exposes the switch. `xmake.lua:6-9` declares:

```lua
option("include_logging")
    set_showmenu(true)
    set_description("Include verbose logging on run")
    add_defines("VERBOSE")
```

and both macOS targets, `doorstop_x86_64` and `doorstop_arm64`, call `add_options("include_logging")`.

---

## File Structure

In `dobbo-ca/unitydoorstop`:

| File | Responsibility |
|---|---|
| `src/nix/plthook/plthook_osx.c` | Modify lines 49-52. Put the four `PLTHOOK_DEBUG_*` defines behind a build option so a release binary is quiet. |
| `src/nix/entrypoint.c` | Read only in tasks 1-4. Holds the four hook calls whose failure is currently silent. |
| `xmake.lua` | Modify. Add a `plthook_debug` option, so plthook noise and doorstop logging are independent switches. |

In `dobbo-ca/bepinex` (this repository):

| File | Responsibility |
|---|---|
| `docs/superpowers/plans/2026-09-18-macos-arm64-native-plugin-loading.md` | This plan. |
| `scripts/macos-arm64/run-native.sh` | Create. Launches Valheim native with a named doorstop build and captures both log streams to a known path. |
| `scripts/macos-arm64/check-plugin-loaded.sh` | Create. The test. Exits 0 when a plugin loaded, 1 when it did not. |

---

### Task 1: A harness that fails

The bug has no automated reproduction. Build one first, so every later change is measured rather than eyeballed.

**Files:**
- Create: `scripts/macos-arm64/run-native.sh`
- Create: `scripts/macos-arm64/check-plugin-loaded.sh`

**Interfaces:**
- Consumes: nothing.
- Produces: `run-native.sh <arch>` where `<arch>` is `arm64` or `x86_64`; writes `/tmp/doorstop-probe/<arch>.stderr` and leaves `$V/BepInEx/LogOutput.log` in place. `check-plugin-loaded.sh` takes no arguments, reads `$V/BepInEx/LogOutput.log`, exits 0 when a plugin loaded and 1 otherwise.

- [ ] **Step 1: Write the check script, which is the test**

```bash
#!/bin/sh
# Exits 0 when BepInEx loaded at least one plugin, 1 when it did not.
V="/Users/christopherdobbyn/Library/Application Support/Steam/steamapps/common/Valheim"
LOG="$V/BepInEx/LogOutput.log"

if [ ! -f "$LOG" ]; then
    echo "FAIL: $LOG does not exist -- BepInEx never started logging"
    exit 1
fi

if grep -q "Loading \[" "$LOG"; then
    echo "PASS: plugins loaded"
    grep "Loading \[" "$LOG"
    exit 0
fi

echo "FAIL: no 'Loading [' line in $LOG"
exit 1
```

- [ ] **Step 2: Write the launcher**

`arch` is passed to `ARCHPREFERENCE`. The stock `run_bepinex.sh` is copied, never edited, because a Steam update restores it.

```bash
#!/bin/sh
# Usage: run-native.sh arm64|x86_64
set -e
ARCH="${1:?usage: run-native.sh arm64|x86_64}"
V="/Users/christopherdobbyn/Library/Application Support/Steam/steamapps/common/Valheim"
OUT="/tmp/doorstop-probe"
mkdir -p "$OUT"

# A copy, so a Steam update cannot lose the edit and so the stock script
# stays byte-identical to upstream for comparison.
sed "s/export ARCHPREFERENCE=.*/export ARCHPREFERENCE=\"$ARCH\"/" \
    "$V/run_bepinex.sh" > "$V/run_bepinex_probe.sh"
chmod +x "$V/run_bepinex_probe.sh"

pkill -if "Valheim.app/Contents/MacOS" 2>/dev/null || true
sleep 3
rm -f "$V/BepInEx/LogOutput.log"

cd "$V"
nohup ./run_bepinex_probe.sh ./Valheim.app > "$OUT/$ARCH.stderr" 2>&1 &
echo "launched $ARCH, stderr -> $OUT/$ARCH.stderr"
```

- [ ] **Step 3: Run the harness on x86_64 and confirm it PASSES**

```bash
chmod +x scripts/macos-arm64/*.sh
scripts/macos-arm64/run-native.sh x86_64
# Wait for the main menu. 60 seconds is enough on this machine.
until grep -q "Loaded localization file #12" \
    "/Users/christopherdobbyn/Library/Application Support/Steam/steamapps/common/Valheim/BepInEx/LogOutput.log" 2>/dev/null
do sleep 3; done
scripts/macos-arm64/check-plugin-loaded.sh
```

Expected: `PASS: plugins loaded`, listing at least `Loading [Dobbo Tracker Client 0.8.5]`. If this fails, the harness is wrong, not the game. Fix it before going on.

- [ ] **Step 4: Run the harness on arm64 and confirm it FAILS**

```bash
scripts/macos-arm64/run-native.sh arm64
sleep 60
scripts/macos-arm64/check-plugin-loaded.sh
```

Expected: `FAIL: ... does not exist -- BepInEx never started logging`. That is the bug, now reproducible on demand.

- [ ] **Step 5: Commit**

```bash
git add scripts/macos-arm64/
git commit -m "test(macos): reproducible harness for the arm64 plugin-loading failure"
```

---

### Task 2: Build Doorstop 4.5.0 from source and prove the build is equivalent

Before changing doorstop, prove that a locally built 4.5.0 behaves exactly like the shipped binary. Without this, a later behaviour change cannot be attributed to the source edit.

**Files:**
- Modify: none. This task builds `dobbo-ca/unitydoorstop` at tag `v4.5.0`.

**Interfaces:**
- Consumes: `scripts/macos-arm64/run-native.sh` and `check-plugin-loaded.sh` from Task 1.
- Produces: a universal `libdoorstop.dylib` at a path recorded in the commit message, plus a saved copy of the shipped binary at `$V/libdoorstop.dylib.stock`.

- [ ] **Step 1: Keep the shipped binary**

```bash
V="/Users/christopherdobbyn/Library/Application Support/Steam/steamapps/common/Valheim"
cp "$V/libdoorstop.dylib" "$V/libdoorstop.dylib.stock"
lipo -archs "$V/libdoorstop.dylib.stock"
```

Expected: `x86_64 arm64`.

- [ ] **Step 2: Get the source at the matching tag**

```bash
cd ~/work/dobbo-ca
git clone git@github.com:dobbo-ca/unitydoorstop.git 2>/dev/null || true
cd unitydoorstop
git checkout -b arm64-diagnostics-4d1c v4.5.0
```

- [ ] **Step 3: Install xmake and build both slices**

```bash
brew install xmake
cd ~/work/dobbo-ca/unitydoorstop
xmake f -m release -y
xmake build doorstop_x86_64
xmake build doorstop_arm64
find build -name "libdoorstop*.dylib" -o -name "*.dylib" | head
```

`xmake.lua:104-135` builds `doorstop_arm64` and then lipos both slices into a `universal` directory. Note: its `after_build` calls `os.execv("sleep", {"5"})` to wait for the sibling build, so building the two targets in the wrong order or in parallel can produce a universal binary with one stale slice. Always run the two `xmake build` commands in the order above and check the result.

- [ ] **Step 4: Verify the built binary is universal**

```bash
lipo -archs $(find ~/work/dobbo-ca/unitydoorstop/build -path "*universal*" -name "*.dylib" | head -1)
```

Expected: `x86_64 arm64`. If only one architecture appears, the sleep race bit; rebuild both targets and check again.

- [ ] **Step 5: Install the local build and re-run both harness arms**

```bash
V="/Users/christopherdobbyn/Library/Application Support/Steam/steamapps/common/Valheim"
cp $(find ~/work/dobbo-ca/unitydoorstop/build -path "*universal*" -name "*.dylib" | head -1) "$V/libdoorstop.dylib"

cd ~/work/dobbo-ca/bepinex
scripts/macos-arm64/run-native.sh x86_64 && sleep 60 && scripts/macos-arm64/check-plugin-loaded.sh
scripts/macos-arm64/run-native.sh arm64   && sleep 60 && scripts/macos-arm64/check-plugin-loaded.sh
```

Expected: x86_64 PASS, arm64 FAIL — identical to Task 1. If the local build changes either result, stop. The build differs from the shipped one, and that difference must be understood before any diagnosis built on it.

- [ ] **Step 6: Commit the finding**

In `dobbo-ca/bepinex`:

```bash
git commit --allow-empty -m "chore(macos): doorstop 4.5.0 built from source reproduces both harness arms

Local universal build behaves identically to the shipped binary:
x86_64 loads plugins, arm64 does not. Any behaviour change after
this point is attributable to a source edit."
```

---

### Task 3: Separate doorstop's diagnostics from plthook's noise

`plthook_osx.c` writes about 7,100 lines per launch in a release build while doorstop's own messages are compiled out. Turning on `VERBOSE` without silencing plthook would bury the signal in that dump.

**Files:**
- Modify: `~/work/dobbo-ca/unitydoorstop/src/nix/plthook/plthook_osx.c:49-52`
- Modify: `~/work/dobbo-ca/unitydoorstop/xmake.lua:6-9`

**Interfaces:**
- Consumes: the branch `arm64-diagnostics-4d1c` from Task 2.
- Produces: a build option `plthook_debug`, off by default. With it off and `include_logging` on, a run writes `[Doorstop]` lines and no bind-opcode dump.

- [ ] **Step 1: Put the plthook debug macros behind the new option**

Replace lines 49-52 of `src/nix/plthook/plthook_osx.c`:

```c
/* Was four unconditional `#define PLTHOOK_DEBUG_* 1` lines, which made a
   RELEASE build write ~7100 lines of bind opcodes to stderr on every
   launch. Now opt-in, set by the `plthook_debug` xmake option. */
#ifdef PLTHOOK_DEBUG
#define PLTHOOK_DEBUG_CMD 1
#define PLTHOOK_DEBUG_BIND 1
#define PLTHOOK_DEBUG_FIXUPS 1
#define PLTHOOK_DEBUG_ADDR 1
#endif
```

- [ ] **Step 2: Add the option to the build**

After the `include_logging` option block in `xmake.lua` (lines 6-9), add:

```lua
option("plthook_debug")
    set_showmenu(true)
    set_description("Dump Mach-O bind opcodes from plthook (very noisy)")
    add_defines("PLTHOOK_DEBUG")
```

Then add `add_options("plthook_debug")` to the `doorstop_x86_64` target beside its existing `add_options("include_logging")` line, and the same to `doorstop_arm64`.

- [ ] **Step 3: Build quiet, with doorstop logging on**

```bash
cd ~/work/dobbo-ca/unitydoorstop
xmake f -m release --include_logging=y --plthook_debug=n -y
xmake build doorstop_x86_64
xmake build doorstop_arm64
```

- [ ] **Step 4: Verify the noise is gone and the signal is present, on the arm that works**

Install as in Task 2 Step 5, then:

```bash
cd ~/work/dobbo-ca/bepinex
scripts/macos-arm64/run-native.sh x86_64
sleep 60
grep -c "BIND_OPCODE" /tmp/doorstop-probe/x86_64.stderr
grep -c "\[Doorstop\]" /tmp/doorstop-probe/x86_64.stderr
scripts/macos-arm64/check-plugin-loaded.sh
```

Expected: `BIND_OPCODE` count **0**, `[Doorstop]` count **greater than 0**, and still `PASS: plugins loaded`. The working arm must keep working, or the edit broke something.

- [ ] **Step 5: Commit**

```bash
cd ~/work/dobbo-ca/unitydoorstop
git add src/nix/plthook/plthook_osx.c xmake.lua
git commit -m "build(nix): make plthook's bind-opcode dump opt-in

plthook_osx.c hard-defined four PLTHOOK_DEBUG_* macros with no guard, so
release builds wrote ~7100 lines of Mach-O bind opcodes to stderr on every
launch, while doorstop's own LOG() calls were compiled out entirely. The
two are now independent build options."
```

---

### Task 4: Capture the failure

This is the task the whole plan exists for. It changes no product code.

**Files:**
- Create: `docs/superpowers/plans/2026-09-18-arm64-findings.md` in `dobbo-ca/bepinex`

**Interfaces:**
- Consumes: the quiet, verbose doorstop build from Task 3.
- Produces: a findings document naming the exact call that fails and its `plthook_error()` string.

- [ ] **Step 1: Run the failing arm with diagnostics on**

```bash
cd ~/work/dobbo-ca/bepinex
scripts/macos-arm64/run-native.sh arm64
sleep 60
grep "\[Doorstop\]" /tmp/doorstop-probe/arm64.stderr
```

- [ ] **Step 2: Compare the two arms line by line**

```bash
diff <(grep "\[Doorstop\]" /tmp/doorstop-probe/x86_64.stderr) \
     <(grep "\[Doorstop\]" /tmp/doorstop-probe/arm64.stderr)
```

The first line present on x86_64 and absent on arm64 is the failure point.

- [ ] **Step 3: Read the surrounding source**

The candidates, all in `src/nix/entrypoint.c`, in the order they run:

| Line | Call | Message when it fails |
|---|---|---|
| 106 | `plthook_handle_by_name("UnityPlayer")` | none; falls through to `plthook_open` |
| 111 | `plthook_open(&hook, NULL)` | `Failed to open current process PLT! Cannot run Doorstop!` and **returns** |
| 119 | `plthook_replace(hook, "dlsym", ...)` | `Failed to hook dlsym, ignoring it.` and continues |
| 147 | `plthook_replace(hook, "fclose", ...)` | `Failed to hook fclose, ignoring it.` |
| 151 | `plthook_replace(hook, "dup2", ...)` | `Failed to hook dup2, ignoring it.` |
| 164 | `plthook_replace(hook, "mono_jit_init_version", ...)` | `Failed to hook jit_init_version, ignoring it.` |

Line 111 is the only one that aborts. The rest log and carry on, which is how a run can end with no plugins and no crash.

- [ ] **Step 4: Write the findings document**

Record, with the exact quoted output: which line was last reached, the `plthook_error()` string, and whether `dlsym_hook` was ever entered. Do not propose a fix in this document. It records what was observed.

- [ ] **Step 5: Commit**

```bash
cd ~/work/dobbo-ca/bepinex
git add docs/superpowers/plans/2026-09-18-arm64-findings.md
git commit -m "docs(macos): record where doorstop fails on arm64"
```

---

### Task 5: DECISION GATE — do not plan past here

Stop. The fix cannot be specified before Task 4 produces the failing call, and inventing steps for an unknown cause is how a plan becomes fiction.

When Task 4 lands, re-run the brainstorming skill against its findings and append Tasks 6 onward to this file. Three outcomes are worth anticipating, because they differ by an order of magnitude in cost:

1. **A hook fails and reports a clear reason.** Likely a bounded fix in `plthook_osx.c`. Hours.
2. **Every hook succeeds and the preloader still never logs.** The failure has moved into managed code, and `BepInEx.Preloader` is the next place to instrument. Days.
3. **A hook succeeds but the patched code never runs.** Symbol binding differs under arm64 in a way `plthook` does not model. This is the expensive branch, and it is the point at which abandoning the effort is a reasonable answer.

Whatever the branch, note that a working chainloader is necessary but not sufficient: MonoMod and Harmony must also emit and apply detours under arm64 Mono. That is a second, untested unknown, and no task in this plan touches it.

---

### Task 6: Upstream the diagnostics defect

Independent of whether arm64 is ever fixed. A release build that writes 7,100 lines of debug output while suppressing its own error messages is a defect on its own terms, and the change from Task 3 is small, self-contained and useful to everyone.

**Files:**
- Modify: none. This task opens a pull request from work already committed in Task 3.

**Interfaces:**
- Consumes: the commit from Task 3 Step 5.
- Produces: a pull request against `NeighTools/UnityDoorstop`.

- [ ] **Step 1: Push the branch to the fork**

```bash
cd ~/work/dobbo-ca/unitydoorstop
git branch --show-current   # must be arm64-diagnostics-4d1c
git push -u origin arm64-diagnostics-4d1c
```

- [ ] **Step 2: Open the pull request against upstream, explicitly**

`gh pr create` in a fork targets upstream by default, which here is what is wanted — but state it, so the next reader does not have to know that.

```bash
gh pr create --repo NeighTools/UnityDoorstop \
  --base master --head dobbo-ca:arm64-diagnostics-4d1c \
  --title "build(nix): make plthook's bind-opcode dump opt-in" \
  --body "plthook_osx.c hard-defines PLTHOOK_DEBUG_CMD, _BIND, _FIXUPS and _ADDR to 1 with no guard, so a release build writes roughly 7100 lines of Mach-O bind opcodes to stderr on every launch. Doorstop's own LOG() calls are compiled out in the same build, because logging.h only defines them under VERBOSE. The result is that a release binary is loud about routine work and silent about its own failures. This adds a plthook_debug option, off by default, leaving include_logging untouched."
```

- [ ] **Step 3: Confirm the pull request targets the right repository**

```bash
gh pr view --repo NeighTools/UnityDoorstop --json baseRefName,headRefName,url
```

Expected: base `master`, head `arm64-diagnostics-4d1c`, and a URL under `NeighTools/UnityDoorstop`. If the URL is under `dobbo-ca`, the pull request went to the fork; close it and redo Step 2.
