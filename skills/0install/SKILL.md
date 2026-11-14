---
name: 0install
description: >-
  Run a command-line tool with Zero Install (`0install run <feed-uri>`) without
  installing it, leaving PATH, package managers and system directories
  untouched. Use when a tool you need (a runtime like Node.js, a utility like
  jq or 7-Zip, a build tool) is not installed and you would otherwise have to
  install it, or when you need a specific version alongside what's already on
  the system. Works even when 0install itself is not installed.
---

# Running command-line tools with 0install

[Zero Install](https://0install.net/) (`0install`) downloads and runs programs,
together with all their dependencies, directly from the web. Each version of
each package is unpacked into its own directory in a per-user cache
(`~/.cache/0install.net/` on Linux/macOS, `%LocalAppData%\0install.net\` on
Windows). It doesn't write to `/usr/bin`, the registry, `PATH` or the Start
menu, and doesn't run install scripts as root. When you only need a tool to
complete a task, this means there is no install step to undo, no version
conflict with the user's existing packages, and nothing left behind that the
user didn't ask for.

**Platform coverage varies.** Feeds on apps.0install.net generally have
up-to-date binaries for Windows. Many also have them for Linux and macOS (e.g.
Node.js, jq), but some only offer old builds there or defer to the
distribution's own package. Check what you'd get with `0install select` before
relying on a specific version (see below).

**There are two implementations with slightly different command lines:** the
.NET one on Windows and the OCaml one on Linux/macOS. Differences are called
out below as "Windows" and "Linux/macOS".

## Workflow

1. **Get a working `0install` command** (use what's on PATH, otherwise
   bootstrap; see below).
2. **Find the tool's feed URI** with `0install search` (on Windows, try
   `0install catalog search` first).
3. **Run the tool** with `0install run <feed-uri> [args]`, adding `--batch`
   (Windows) or `--console` (Linux/macOS) so it never waits on a prompt or
   window you can't see.

## 1. Get a working `0install` command

If `0install` is on `PATH` (`command -v 0install` / `Get-Command 0install`),
use it directly.

Otherwise use the bootstrap scripts bundled with this skill:
[`scripts/0install.sh`](scripts/0install.sh) and
[`scripts/0install.ps1`](scripts/0install.ps1), next to this file. These are
tiny: they use `0install` from `PATH` if present, otherwise download a full
0install into the per-user cache on first use, and forward all arguments to it,
so `0install.sh run ...` behaves exactly like `0install run ...`. Nothing is
installed system-wide.

Run them in place, using the absolute path of this skill's directory. Don't
copy them into the user's project.

**Linux / macOS** (needs `curl`, `tar` and `bzip2` for the first-use download):

```bash
sh <skill-dir>/scripts/0install.sh run --console <feed-uri> [args]
```

Prebuilt 0install binaries are only guaranteed for x86_64. On Apple Silicon the
script downloads the x86_64 build, which needs Rosetta 2. On other
architectures the download may fail; if it does, tell the user rather than
working around it.

**Windows**:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File <skill-dir>\scripts\0install.ps1 run --batch <feed-uri> [args]
```

`-ExecutionPolicy Bypass` avoids "running scripts is disabled on this system"
errors; it only applies to this one invocation.

On Windows always use the PowerShell bootstrap, even when your shell is Git
Bash or similar (call it through `powershell.exe`). `0install.sh` doesn't
support Windows.

Below, `0install` means "the command you resolved in this step": the binary on
`PATH`, `sh <skill-dir>/scripts/0install.sh`, or the
`powershell ... -File <skill-dir>\scripts\0install.ps1` invocation.

## 2. Find the tool's feed URI

Programs are identified by the URL of their **feed** (an XML file), e.g.
`https://apps.0install.net/javascript/node.xml`. To discover the feed for a
tool, search by keyword.

**Windows: try `0install catalog search` first.** It searches the curated
catalog of apps.0install.net, so a hit is a maintained feed. This subcommand
only exists on Windows.

```powershell
0install catalog search jq
```

**All platforms: `0install search`** queries the mirror server, which indexes
essentially every known feed. Use it on Linux/macOS, and on Windows when the
catalog has no match.

```bash
0install search jq
```

Both print matching feed URIs. Pick the one whose name/description matches the
tool you want. `0install search` indexes feeds regardless of quality, so
sanity-check that a result is the genuine project before running it; prefer
feeds hosted on `apps.0install.net` or on the project's own domain.

You can also list feeds you've already used locally:

```bash
0install list           # all locally known feeds
0install list node      # filter by substring
```

## 3. Run the tool

```bash
0install run [options] <feed-uri> [program args...]
```

The first run downloads and caches the program and its dependencies, then
executes it; later runs reuse the cache and start immediately.

The exit status is the program's own exit status, or `1` if 0install couldn't
select or download it. A `1` is therefore ambiguous: read stderr to tell a
0install failure from the program itself failing.

Examples (add `--batch` or `--console` as described below):

```bash
# Run Node.js
0install run https://apps.0install.net/javascript/node.xml --version

# Run npm, a second command in the same feed
0install run --command npm https://apps.0install.net/javascript/node.xml install

# Pick a Node.js major version
0install run --version 22..!23 https://apps.0install.net/javascript/node.xml --version

# Run jq
0install run https://apps.0install.net/utils/jq.xml --help
```

Useful options for `run` (also valid for `select` / `download`). Options go
**before** the feed URI; everything after it is passed to the program.

| Option | Purpose |
| --- | --- |
| `--batch` | (Windows only) Don't ask questions; plain console output. Can't be combined with `--verbose`. |
| `--console` / `-c` | (Linux/macOS only) Never open a GUI window; use the terminal. |
| `--version RANGE` | Require a version range of the program, e.g. `--version 22..!23`. Ranges are `START..!END` (END exclusive, either may be omitted), joined with `\|`. A bare version such as `2.7` matches **only** exactly 2.7, not 2.7.1; use `2.7..!2.8` for a release line. `!VERSION` excludes one version. |
| `--not-before VERSION` | Require at least the given version. |
| `--before VERSION` | Require a version earlier than the given one. |
| `--version-for URI RANGE` | Restrict the version of a dependency, e.g. the runtime a tool uses. |
| `--command COMMAND` | Run a non-default command defined by the feed (e.g. `npm` from the Node.js feed). |
| `--refresh` | Re-fetch feeds to pick up the latest versions before running. |
| `--offline` | Use only what's already cached; don't hit the network. |
| `-v` / `--verbose` | More detail (use twice for even more). On Windows, drop `--batch` when using this. |

Related commands you may want instead of `run`:

- `0install select <feed-uri>`: show which versions *would* be used (and where
  they're cached on disk) without running anything. Use this to check that a
  feed has a suitable version for this platform before relying on it.
- `0install download <feed-uri>`: pre-fetch into the cache without running,
  e.g. to warm the cache before going offline.

### Prompts: key trust, system packages and GUI windows

You usually can't answer interactive prompts or see GUI windows, so always
pass the non-interactive option for your platform to `run` / `select` /
`download`:

- **Windows: `--batch`.** Answers questions with safe defaults where possible
  and gives plain console output (no progress bars or tables), which is also
  easier to read.
- **Linux/macOS: `--console`.** Without it, 0install may open a graphical
  window whenever `$DISPLAY` is set, and the command then hangs waiting for
  input you can't give.

Two kinds of question can still come up:

- **Key trust.** Feeds are GPG-signed. The first time 0install sees a key for
  a given domain, it trusts it automatically if the key information server
  knows the key (usually the case for well-known feeds). Otherwise it asks for
  confirmation.
- **Distribution packages (Linux).** When a feed prefers the system's own
  package, 0install may offer to install it via PackageKit. Unlike everything
  else `0install run` does, this *does* change the system. Separately, if a
  suitable version is already installed through the OS package manager,
  0install may simply use that instead of downloading its own copy.

If a command stops on a key-trust or package-install prompt, or fails because
it needed one, **stop and ask the user** whether to accept it. Don't try to
answer the prompt yourself.

### Optional: a persistent shortcut

If the user explicitly wants a tool available long-term as a normal command,
you can add it to their app list with a command name. This creates a launcher
on `PATH`, so it is no longer "leave no trace":

```bash
0install add jq https://apps.0install.net/utils/jq.xml
jq --version
```

Always pass the name. It is required on Linux/macOS; on Windows it is
optional, but without it no launcher is created.

To undo it:

```bash
0install destroy jq    # Linux/macOS
0install remove jq     # Windows
```

For the clean, leave-no-trace use case, stick with `0install run`.

## Further reading

The Zero Install docs have the full
[command-line reference](https://docs.0install.net/details/cli/) and a
[getting started](https://docs.0install.net/basics/) guide.
