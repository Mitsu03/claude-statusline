# Windows (Git Bash) setup

The script targets macOS/Linux. It runs on Windows under Git Bash, but needs the
fixes described below — all of them are already applied in
`statusline-command.sh` and are no-ops on macOS/Linux.

## Setup

1. Copy the script into your Claude config directory:

   ```bash
   cp statusline-command.sh ~/.claude/statusline-command.sh
   ```

2. Point `~/.claude/settings.json` at it **through Git Bash's absolute path**:

   ```json
   {
     "statusLine": {
       "type": "command",
       "command": "\"C:\\Program Files\\Git\\bin\\bash.exe\" \"C:/Users/<you>/.claude/statusline-command.sh\""
     }
   }
   ```

   Do not use the README's `bash ~/.claude/statusline-command.sh`. On a default
   Windows install, `bash` on the system PATH resolves to
   `C:\WINDOWS\system32\bash.exe`, which is **WSL** — a different filesystem with
   its own `$HOME` and no access to the tools this script expects. The status line
   either renders nothing or renders from the wrong home directory.

## Dependencies

- `jq` — required. `winget install jqlang.jq`.
- `bc` — **not** required here. Git Bash ships without it, so the script defines
  an `awk`-backed fallback (see below). A real `bc`, if present, still wins.
- Everything else (`awk`, `sed`, `stat`, `date`, `cksum`, `mktemp`) ships with
  Git for Windows.

## The fixes

### 1. `bc` does not exist in Git Bash

The script uses `bc` in ten places — pace projection, burn rate, cost per 1k
tokens, budget tracking. Without it, every one of those fields renders empty.

A `bc()` shell function now stands in when no real `bc` is on PATH. It covers the
subset used here (`scale=N; <expr>` and plain arithmetic/comparisons) by
interpolating the expression into an `awk` program — the shell has already
expanded the variables to numbers by that point, so `awk` parses and evaluates
it. It is defined only behind `command -v bc`, so a real `bc` takes precedence.

### 2. `sed -i ''` is BSD-only

GNU sed reads the empty string as a filename and fails, breaking the `budget`,
`usage` and `sync` subcommands. Replaced with a `sed_inplace()` helper that
writes to a sibling temp file and swaps it in — portable, and atomic as a bonus.

### 3. `stat -f %m` writes to stdout on failure

The mtime lookup was `stat -f %m … || stat -c %Y …`, BSD first. On GNU stat,
`-f` means *filesystem info*: it prints a multi-line filesystem report **to
stdout** and then exits 1. The `||` fallback runs, but the junk from the first
command is already in the captured output, so `last_ts` becomes garbage and the
"last interaction" clock never renders.

Reordered to try `stat -c %Y` first, with `-f %m` as the fallback. BSD stat
rejects an unknown `-c` cleanly on stderr, so macOS still resolves on the second
try.

### 4. `~` shortening never matched the cwd

Claude Code hands the script a **native** path — `C:\Users\me\project` — while
Git Bash's `$HOME` is `/c/Users/me`. The `${cwd/#$HOME/~}` prefix strip therefore
never matched and line 3 showed the full Windows path.

A `norm_path()` helper now puts both sides in the same shape (forward slashes,
`C:/…` drive form) before comparing, and the comparison is case-insensitive
because Windows paths are. The drive rewrite is gated on `$OSTYPE` so it stays a
no-op on macOS/Linux, where turning `/a/b` into `a:/b` would corrupt real paths.

| cwd received | line 3 |
| --- | --- |
| `C:\Users\me\Documents\proj` | `~/Documents/proj` |
| `C:\Users\me` | `~` |
| `c:\users\ME\Documents` | `~/Documents` |
| `C:\Users\me2\x` | `C:/Users/me2/x` (not shortened — similar prefix, different user) |
| `D:\Projects\foo` | `D:/Projects/foo` |

### 5. The transcript id leaked the whole path and corrupted the output

Same root cause as #4, but with two consequences instead of one.
`transcript_file="${transcript_path##*/}"` strips nothing when the path uses
backslashes, so the entire path reached the closing `printf "%b"` — which then
interpreted the backslashes as escapes:

```
~/Documents/proj │ C:\Users\me\.claude\projects\…-proj db24598-7f19-4002-…
statusline-command.sh: line 516: printf: missing unicode digit for \U
```

`\Users` tripped the `\U` unicode escape and emitted an error to stderr on every
render, and `\0db24598…` was swallowed as an octal escape — note the missing `0`.

Normalizing `transcript_path` before the basename strip fixes both. It also
fixes the **usage line**: `[ -f "$transcript_path" ]` and `stat` were failing on
the backslash path, so the last-interaction clock never appeared either.

## Known remaining rough edge

`fmt_clock()` calls `date -r "$ts"` (BSD: epoch) before `date -d "@$ts"` (GNU).
GNU `date -r` means *reference file*, so the first call fails — but it fails
cleanly, writing nothing to stdout, so the fallback produces the right answer.
Left as-is: it costs one wasted `date` invocation per render and changing the
order would only move the wasted call to macOS.
