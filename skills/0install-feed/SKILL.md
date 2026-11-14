---
name: 0install-feed
description: >-
  Write a Zero Install (0install) feed for a program: author the feed template
  (`.xml.template`), generate the feed for a release with 0template, then lint
  and test-run it locally. Use when the user wants to package an app, tool or
  library for 0install (e.g. "create a 0install feed for my app", "make this
  runnable with 0install run", "add an arm64 build to the feed", "the feed's
  archive layout changed") or wants a local feed to run a source checkout.
  For signing and hosting the result, see the `0install-publish` skill.
---

# Writing a Zero Install feed

A [Zero Install](https://0install.net/) **feed** is an XML file that describes
a program: its metadata, its versions ("implementations"), where to download
each one, how to start it and what it depends on. Users run it with
`0install run <feed-url>`.

You rarely write a feed by hand. Instead you write a **template** once and let
[0template](https://docs.0install.net/tools/0template/) stamp out the feed for
each release:

```
myapp.xml.template ──(0template version=1.2)──▶ myapp-1.2.xml ──(0publish)──▶ myapp.xml
  feed with {holes}                              one version              master feed, signed
```

- The **template** is a feed with `{placeholder}` holes and an empty
  `<manifest-digest/>`.
- **0template** fills the holes, downloads each archive, and computes its
  size and digest. The result is a **per-version feed**, a throwaway
  intermediate.
- The **master feed** at the permanent URL lists all versions. Building,
  signing and hosting it is covered by the `0install-publish` skill.

This skill covers everything up to a per-version feed that lints cleanly and
runs.

## Conventions

- `0install` below means a working 0install command. If it isn't on `PATH`,
  use the bootstrap scripts of the `0install` skill, or tell the user to get
  it from <https://get.0install.net/>.
- The tools are run through 0install, so nothing needs installing:
  `0install run https://apps.0install.net/0install/0template.xml ...`. Add
  `--batch` (Windows) or `--console` (Linux/macOS) after `run` so it never
  waits on a prompt you can't see.
- Never write `id`, `size` or digest values yourself, and never edit them in
  an existing feed. 0template computes them.

## Step 1: Gather the facts

Find these before writing anything. Ask the user only for what you can't
determine from the project or its release page.

- **Identity**: name, one-line summary, longer description, homepage, license
  as an SPDX identifier (`MIT`, `Apache-2.0`, `GPL-3.0-or-later`).
- **Feed URL**: the permanent URL the master feed will be served from, e.g.
  `https://OWNER.github.io/myapp/myapp.xml`. It goes into the template and
  **must never change** afterwards, so settle it with the user now. (Not
  needed for a local feed.)
- **Kind**: a runnable app or tool (has a `<command name="run">`), or a
  library (no command; other feeds consume it).
- **Runtime**: a native binary, or something that needs a runtime (.NET,
  Java, Python, ...)? Runtimes are pulled in as a `<runner>`; see
  [references/runtimes.md](references/runtimes.md).
- **Downloads**: the exact URL pattern of the release archives, and which
  OS/CPU builds exist. Archives must be reachable over HTTP(S) and must never
  be replaced once published: the feed pins their digest.
- **CLI or GUI**: command-line tools get `<needs-terminal/>`, GUI apps don't.

## Step 2: Write the template

Name it `<name>.xml.template`, where `<name>.xml` is the file name of the
feed URL. A cross-platform native app:

```xml
<?xml version="1.0" encoding="utf-8"?>
<interface xmlns="http://zero-install.sourceforge.net/2004/injector/interface">
  <name>MyApp</name>
  <summary>one line, lowercase start, no trailing period</summary>
  <description>A longer description.</description>
  <homepage>https://github.com/OWNER/myapp</homepage>
  <needs-terminal/>
  <icon href="https://OWNER.github.io/myapp/myapp.png" type="image/png"/>

  <feed-for interface="https://OWNER.github.io/myapp/myapp.xml"/>

  <group license="MIT">
    <command name="run" path="bin/myapp"/>
    <implementation arch="Linux-x86_64" version="{version}" released="{released}" stability="stable">
      <manifest-digest/>
      <archive href="https://github.com/OWNER/myapp/releases/download/v{version}/myapp-{version}-linux-x64.tar.gz"/>
    </implementation>
    <implementation arch="Darwin-aarch64" version="{version}" released="{released}" stability="stable">
      <manifest-digest/>
      <archive href="https://github.com/OWNER/myapp/releases/download/v{version}/myapp-{version}-macos-arm64.tar.gz"/>
    </implementation>
  </group>

  <group license="MIT">
    <command name="run" path="bin/myapp.exe"/>
    <implementation arch="Windows-x86_64" version="{version}" released="{released}" stability="stable">
      <manifest-digest/>
      <archive href="https://github.com/OWNER/myapp/releases/download/v{version}/myapp-{version}-windows-x64.zip"/>
    </implementation>
  </group>
</interface>
```

Rules that matter:

- **`<feed-for>`** holds the permanent feed URL. It tells the publishing
  tools which master feed a per-version feed belongs to.
- **`<manifest-digest/>` stays empty**, with no `id` or `size` anywhere.
- **Placeholders**: `{version}` and usually `{released}` (`YYYY-MM-DD`). You
  may invent more (e.g. `{version-original}` when the download URL uses a raw
  tag that differs from the normalized version). Every `{key}` must get a
  value when 0template runs, and every value must be used by some `{key}`.
- **Versions** are dotted numbers with optional `-pre`, `-rc` or `-post`
  modifiers: `1.2.3`, `1.2-rc1`, `2.0-post1`. Not `v1.2.3`, `1.2rc1` or
  `1.2.3a`.
- **`<archive>`** is for `.zip`/`.tar.*` and similar. If the archive wraps
  everything in one top-level directory, 0template detects it and sets
  `extract`. `<command path>` is relative to that directory. Don't guess the
  layout: list the archive (`tar -tf`, `unzip -l`) and look.
- **`<file dest="myapp" href="..."/>`** is for a single downloaded file (a
  bare binary, an `.exe`, a `.jar`). Add `executable="true"` for a Unix
  binary.
- **One `<implementation>` per download**, each with its `arch="OS-CPU"`.
  Implementations that share the same `<command>` share a `<group>`. Where
  the path differs per OS (`.exe`), use a separate top-level `<group>`; don't
  nest groups.
- **Never give two implementations the same archive.** A build that covers
  several platforms gets one implementation with a broader `arch`
  (`Darwin-*`, `*-*`).
- **`<icon>`**: a PNG hosted at a stable URL, usually next to the feed. Leave
  the element out if there is no icon; a dangling URL is worse than none.

Then read what applies:

- [references/feed-elements.md](references/feed-elements.md): arch tokens,
  several commands and `<entry-point>`s, dependencies and bindings, icons,
  letting 0template build the archive from a local directory, splitting a
  feed per OS.
- [references/runtimes.md](references/runtimes.md): recipes for .NET, .NET
  Framework, Java, Python, native Windows/Linux/macOS packages (`.msi`,
  `.deb`, `.rpm`, `.dmg`, AppImage) and libraries.

### Local feed instead of a template

To run a source checkout or build output without any release, write a plain
feed (no `.template`, no `<feed-for>`) next to the files, with `id="."` or
`local-path`:

```xml
<implementation id="." version="0.1-pre">
  <command name="run" path="myapp.py">
    <runner interface="https://apps.0install.net/python/python.xml"/>
  </command>
</implementation>
```

`0install run ./myapp.xml` then runs it straight from that directory,
fetching the dependencies. Nothing is downloaded or digested, and no signature
is needed. Skip step 3.

## Step 3: Generate the feed for one version

The release archives must already be downloadable (or sit next to the
template under the file name their `href` ends with; 0template then uses the
local copy).

```bash
0install run https://apps.0install.net/0install/0template.xml myapp.xml.template version=1.2.3 released=2026-01-31
```

This downloads each archive into the template's directory and writes
`myapp-1.2.3.xml` there. If a download fails or a path is wrong, fix the
template and rerun; delete the stale output first.

Neither the downloaded archives nor the per-version feeds belong in version
control. Add them to `.gitignore` if the template lives in a Git repo.

## Step 4: Verify

**a. Read the generated feed.** Each implementation should have an `id`, a
`manifest-digest` and an archive `size`. Check that `extract` is what you
expect. The digest
`sha256new="4OYMIQUY7QOBJGX36TEJS35ZEQT24QPEMSNZGTFESWMRW6CSXBKQ"` is that of
an **empty directory**: `extract` names a directory the archive doesn't have.

**b. Lint it.**

```bash
0install run https://apps.0install.net/0install/feedlint.xml myapp-1.2.3.xml
```

Fix what it reports, with two exceptions that are not defects in your feed:

- A 404 for the `<icon>` or `<homepage>` when that page isn't published yet.
- A crash with `AttributeError: 'FileSource' object has no attribute
  'start_offset'` on implementations that use `<file>`. Read the output above
  the traceback and rely on (c) for those.

**c. Select and run it.**

```bash
0install select myapp-1.2.3.xml
```

```bash
0install run myapp-1.2.3.xml --version
```

`select` resolves the app and its dependencies without launching anything.
`run` starts it; skip that for libraries, and for GUI apps unless the user
wants a window opened. Only the implementation for the current OS and CPU can
be test-run. For the others, compare each archive listing against `extract`
and the `<command>` path, and say in your report which platforms were not
run.

## Next: publish

A per-version feed only works from the local disk. To make
`0install run https://...` work for everyone, the master feed must be signed
and hosted: continue with the `0install-publish` skill. Keep
`myapp-1.2.3.xml` for it.

## Changing an existing feed

A template only affects versions generated after the change. Published
implementations are immutable.

| Change | What to do |
| --- | --- |
| Upstream renamed files or changed the archive layout | Fix the template, then steps 3 and 4 with the newest release. |
| New OS or CPU build | Add an `<implementation>` to the template. It applies to future versions. |
| New command or dependency | Change the template. Existing versions in the master feed keep what they had. |
| Metadata (summary, description, icon, entry points) | Change the template **and** the master feed. Adding a version copies implementations only, so the master feed keeps its old metadata. The master feed is signed; see `0install-publish` for editing it. |

## Further reading

- [Feed specification](https://docs.0install.net/specifications/feed/): every
  element and attribute.
- [Packaging guides](https://docs.0install.net/packaging/guides/): the full
  per-runtime recipes that `references/runtimes.md` condenses.
