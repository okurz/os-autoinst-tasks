# 013: Pretty Serial Marker Without VNC Typing

## Problem Statement

os-autoinst types commands and a serial marker (hash + exit code) over VNC
character-by-character. This has three drawbacks:

1. **Ugly screenshots/videos** — reviewers see the raw `; echo Ab12x-$?- >
   /dev/ttyS0` marker appended to every command.
2. **Slow** — VNC typing runs at ~30 chars/sec (33ms/key). A 60-char marker
   suffix adds ~2 seconds per command.
3. **Unreliable under load** — character-by-character RFB key events can drop
   keystrokes when the host or guest is under CPU pressure.

## Goal

Automatically forward a unique hash and exit code to the serial port
**without typing it visibly over VNC**. The typed command line should look
clean (just the user's command), while the serial-based synchronization
mechanism continues to work unchanged.

## Current Architecture (relevant files)

| Component | File | Key Lines |
|---|---|---|
| `script_run` (marker construction) | `distribution.pm` | 134-170 |
| `hashed_string` | `testapi.pm` | 1275-1282 |
| `type_string` (VNC char loop) | `consoles/video_base.pm` | 32-61 |
| `wait_serial` | `testapi.pm` | 858-889 |
| `wait_serial` backend | `backend/baseclass.pm` | 878-911 |
| Serial file source (QEMU) | `backend/qemu.pm` | 886-887 |

### Current marker format typed over VNC

```
<cmd>; echo <HASH>-$?- > /dev/ttyS0\n
```

`wait_serial` then matches `qr/<HASH>-\d+-/` in the serial file.

## Proposed Solution

### Core Idea: Shell Trap / PROMPT_COMMAND

Instead of typing the marker as part of every command, install a **shell hook**
that automatically writes the marker to the serial port after each command
completes. Two viable mechanisms:

#### Option A: `PROMPT_COMMAND` (Recommended for bash)

```bash
export __OSAUTOINST_MARKER="<HASH>"
export PROMPT_COMMAND='echo ${__OSAUTOINST_MARKER}-${__OSAUTOINST_RC:=$?}-'\''\
  > /dev/ttyS0; unset __OSAUTOINST_MARKER __OSAUTOINST_RC'
```

Before each command, os-autoinst sets `__OSAUTOINST_MARKER` via a single
short VNC-typed assignment, then types only the command + Enter. After the
command finishes, `PROMPT_COMMAND` fires and writes the marker to serial.

#### Option B: `trap ... DEBUG` + post-command hook

A DEBUG trap can capture the command text; a PROMPT_COMMAND writes the result.
More complex but allows capturing the actual command string.

#### Option C: `fc` with simplified marker in PROMPT_COMMAND

Use `fc -ln -1` to read the last command from history, derive a short marker
from it using lightweight shell builtins (no `md5sum` dependency), and write
it with `$?` to the serial port. The marker only needs to be unique enough
to distinguish from the *previous* command — not cryptographically strong.

**Simplified SUT-side hashing** — instead of `md5sum`, use shell string
operations that are available in any POSIX shell:
```bash
# Normalize: trim whitespace, take first+last 4 chars + length as marker
_oa_marker() {
  local c="${1## }"; c="${c%% }"
  local l=${#c}
  printf '%.4s%d%.4s' "$c" "$l" "${c: -4}" 2>/dev/null \
    || printf '%.4s%d' "$c" "$l"  # fallback for short commands
}
```
For `ls -la /tmp` this yields `ls -11/tmp` — short, deterministic, and
sufficiently unique to distinguish consecutive commands. os-autoinst computes
the same marker by applying identical normalization in Perl:
```perl
sub sut_marker ($cmd) {
    $cmd =~ s/^\s+|\s+$//g;  # trim
    my $l = length($cmd);
    my $head = substr($cmd, 0, 4);
    my $tail = length($cmd) >= 4 ? substr($cmd, -4) : '';
    return "${head}${l}${tail}";
}
```
This avoids external tool dependencies and `fc` output formatting concerns.

**`fc` normalization:** `fc -ln -1` prepends a tab character and preserves
the command as-is. The trim in `_oa_marker` handles this. Multi-line commands
are joined by `fc` with newlines, but our marker only uses head/tail/length
so multi-line is handled naturally.

#### Option D: Hybrid — PROMPT_COMMAND with env-var, `fc` as enhancement

Combine Options A and C: use the env-var approach as primary (reliable),
but also emit an `fc`-derived marker as a secondary signal that os-autoinst
can use for additional validation or debugging.

**Recommendation:** Option A (`PROMPT_COMMAND` with pre-set env var) as the
primary mechanism, with Option C (`fc` + simplified marker) as a Phase 2
enhancement for fully invisible operation. Both share the same
`PROMPT_COMMAND` infrastructure.

## Execution Plan

### Phase 1: Proof of Concept (non-breaking)

**Step 1.1 — Add SUT capability detection and marker installation**

- File: `distribution.pm`
- Add a method `install_serial_marker_hook($self)` that:
  1. **Probes the SUT environment** to determine capability level:
     - Detect shell type: `echo $BASH_VERSION` (bash), `echo $ZSH_VERSION`
       (zsh), or fall back to `$0` / `readlink /proc/$$/exe`.
     - Detect `PROMPT_COMMAND` support (bash 3+, zsh via `precmd`).
     - Detect `fc` availability: `type fc 2>/dev/null`.
     - Detect if history is enabled: `set -o | grep history`.
     - Detect serial device writability: `test -w /dev/$serialdev`.
  2. **Select the best mechanism** based on detected capabilities:
     - **Level 3 (best):** bash + `fc` + history enabled → fully invisible
       `fc`-based marker via `PROMPT_COMMAND` (Phase 2 target).
     - **Level 2 (good):** bash/zsh + `PROMPT_COMMAND`/`precmd` → env-var
       marker approach (Phase 1 primary).
     - **Level 1 (fallback):** no `PROMPT_COMMAND` support → classic inline
       `; echo <HASH>-$?- > /dev/ttyS0` (current behavior).
  3. Store detected level in `$self->{_serial_marker_level}` for use by
     `script_run`.
  4. Type the appropriate `PROMPT_COMMAND` setup into the SUT shell (for
     levels 2-3).
- Gate behind `PRETTY_SERIAL_MARKER` variable (default off). When enabled,
  run detection automatically on first `script_run` call (lazy init).
- The hook must handle the case where `__OSAUTOINST_MARKER` is unset (user
  typed a manual command) — simply do nothing.
- Cache detection results per console (different consoles may have different
  shells).

**Step 1.2 — Modify `script_run` for new path**

- File: `distribution.pm`, around line 147
- When `PRETTY_SERIAL_MARKER` is enabled and console is VNC (not serial
  terminal):
  1. Type `__OSAUTOINST_MARKER=<HASH>\n` (short, ~25 chars, invisible to
     user if prompt redraws, or can be combined on one line).
  2. Type `<cmd>\n` (clean — no marker suffix).
  3. `wait_serial` unchanged — still matches `qr/<HASH>-\d+-/`.
- The marker variable assignment could alternatively be combined:
  `__OSAUTOINST_MARKER=<HASH> <cmd>\n` (prefix syntax, no semicolon needed,
  variable only lives for that command's environment — but PROMPT_COMMAND
  runs in the parent shell, so the var must be exported or set in parent
  scope). Use `export __OSAUTOINST_MARKER=<HASH>; <cmd>` on one line.
- Actually, simplest approach: type on one line:
  ```
  export __OSAUTOINST_MARKER=<HASH>; <cmd>
  ```
  The `export` + marker is ~35 chars but replaces the old ~30-char suffix,
  so net VNC typing is similar. The **visual win** is that the marker no
  longer appears after the command — the `echo ... > /dev/ttyS0` part is
  gone from screenshots.

**Step 1.3 — Handle edge cases**

- `background_script_run`: Currently uses `echo <HASH>-$!-`. Adapt to set
  marker before `( <cmd> )&` and have the hook report `$!` instead of `$?`.
  This is trickier — may keep old mechanism for background commands initially.
- `script_sudo`: Similar adaptation needed.
- `script_output`: Uses echo delimiters around output capture — keep old
  mechanism initially.
- Serial terminal consoles: No change needed (they don't use VNC typing).
- Shells other than bash (sh, zsh, csh): Handled by dynamic capability
  detection — `zsh` uses `precmd`, `sh`/`dash`/`csh` fall back to classic
  inline marker (Level 1).

**Step 1.4 — Write tests**

- File: `t/27-script_run.t` (or new `t/XX-pretty_serial_marker.t`)
- Test that with `PRETTY_SERIAL_MARKER=1`:
  - `type_string` is called with `export __OSAUTOINST_MARKER=...; <cmd>\n`
    (no ` echo ... > /dev/ttyS0` suffix).
  - `wait_serial` still receives the correct marker pattern.
- Test that without the flag, behavior is unchanged.
- Test `install_serial_marker_hook` types the expected PROMPT_COMMAND.
- Mock `query_isotovideo` as existing tests do.

### Phase 2: Refined Approach — Invisible Marker Setting

**Step 2.1 — Use `stty -echo` trick or control characters**

To make even the `export __OSAUTOINST_MARKER=...` invisible:
- Send the assignment via a secondary channel if available (e.g., virtio
  console, SSH side-channel).
- Or: type `export __OSAUTOINST_MARKER=<HASH>` followed by `\r` (carriage
  return without newline) — the shell executes it but it may be overwritten
  by the prompt on the same line. This is fragile.
- Or: Simply accept that `export __OSAUTOINST_MARKER=Ab12x` is much cleaner
  visually than `; echo Ab12x-$?- > /dev/ttyS0`.

**Step 2.2 — Implement `fc`-based fully invisible marker (Level 3)**

Use `fc -ln -1` with the simplified marker function from Option C:
```bash
PROMPT_COMMAND='_rc=$?; if set -o | grep -q "history.*on"; then
  _c=$(fc -ln -1); _c="${_c#	}"; _c="${_c## }"; _c="${_c%% }";
  _l=${#_c}; printf "OA:%.4s%d%.4s-%d-\n" "$_c" "$_l" "${_c: -4}" "$_rc" \
    > /dev/ttyS0; fi'
```

**os-autoinst side** (`distribution.pm`):
```perl
sub sut_marker ($cmd) {
    $cmd =~ s/^\s+|\s+$//g;
    my $l = length($cmd);
    my $head = substr($cmd, 0, 4);
    my $tail = $l >= 4 ? substr($cmd, -4) : '';
    return "OA:${head}${l}${tail}";
}
```

Both sides apply identical normalization (trim whitespace, take
head/length/tail), so the marker matches deterministically. The `OA:` prefix
avoids false matches in other serial output. No external tools needed — pure
shell builtins.

**`fc` edge cases handled:**
- Tab prefix from `fc -ln -1`: stripped by `${_c#\t}`.
- Leading/trailing spaces: stripped.
- Multi-line commands: length and head/tail still deterministic.
- History disabled: guarded by `set -o` check; falls back to Level 2.

**os-autoinst `wait_serial` call:**
```perl
my $marker = sut_marker($cmd);
wait_serial(qr/\Q$marker\E-(\d+)-/, timeout => $timeout);
```

**Step 2.3 — Add `sut_marker` test coverage**

- Parametrized tests for `sut_marker()` with various inputs:
  - Short commands (`ls`), long commands, commands with special chars.
  - Verify Perl output matches expected shell output for each case.
- Integration-style test: mock a shell session, verify `fc`-derived marker
  matches `sut_marker()` output.

### Phase 3: Cleanup and Default Enablement

**Step 3.1** — After sufficient testing in openQA production (via
`PRETTY_SERIAL_MARKER=1` on select job groups), make it the default.
The dynamic capability detection ensures safe fallback on any SUT.

**Step 3.2** — Update `background_script_run`, `script_sudo`,
`script_output` to also use the new mechanism where feasible.

**Step 3.3** — Document the new behavior. Since SUT capability is detected
dynamically, no manual SUT requirements documentation is needed beyond
explaining the `PRETTY_SERIAL_MARKER` variable.

**Step 3.4** — Log the detected capability level per console to
`autoinst-log.txt` for debugging (e.g., `[serial_marker] console 'root-console':
Level 2 (bash PROMPT_COMMAND, no fc history)`).

## Risk Assessment

| Risk | Likelihood | Mitigation |
|---|---|---|
| `PROMPT_COMMAND` not supported (non-bash shell) | Medium | Dynamic detection auto-selects Level 1 fallback |
| SUT overrides `PROMPT_COMMAND` | Low | Use `__OSAUTOINST_` prefix; append rather than replace |
| Marker fires for manual/interactive commands | Low | Only fires when `__OSAUTOINST_MARKER` is set |
| Race condition: prompt fires before command starts | None | Bash guarantees PROMPT_COMMAND runs after command |
| Breaks existing tests | Low | Gated behind opt-in variable; old path unchanged |
| `fc` marker mismatch (SUT vs Perl) | Low | Simplified head/len/tail scheme uses no locale-sensitive ops; test coverage with parametrized cases |
| SUT has no history / no `fc` | Medium | Capability detection demotes to Level 2 or 1 automatically |
| Detection probe itself fails | Low | Probe errors caught; default to Level 1 (classic) |

## Files to Modify

1. `distribution.pm` — `script_run`, `install_serial_marker_hook`,
   `_detect_serial_marker_capability`, `sut_marker`
2. `testapi.pm` — possibly expose hook installation API
3. `t/27-script_run.t` or new test file — test coverage for all 3 levels
4. Documentation (if any user-facing docs exist for `script_run`)

## Success Criteria

- With `PRETTY_SERIAL_MARKER=1`, screenshots show only `<cmd>` without
  `; echo ... > /dev/ttyS0` suffix.
- `assert_script_run` / `script_run` continue to correctly detect exit codes.
- No regression when the feature is disabled (default off initially).
- All existing tests pass unchanged.
- New tests cover the pretty-marker code path.
