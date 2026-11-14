# Feed element reference

Details behind the template in SKILL.md. The
[feed specification](https://docs.0install.net/specifications/feed/) is the
authority on every element and attribute; this file covers what you need most
often plus additiona context and gotchas that the specification doesn't cover.

## Contents

- [Metadata](#metadata)
- [Arch tokens](#arch-tokens)
- [`<archive>` and `<file>`](#archive-and-file)
- [Groups](#groups)
- [Several commands and entry points](#several-commands-and-entry-points)
- [Dependencies and bindings](#dependencies-and-bindings)
- [Icons](#icons)
- [Letting 0template build the archive](#letting-0template-build-the-archive)
- [One feed per OS](#one-feed-per-os)

## Metadata

Conventional order: `<name>`, `<summary>`, `<description>`, `<homepage>`,
`<category>`, `<needs-terminal/>`, `<icon>`, `<feed-for>` (templates only),
`<entry-point>`s, then the `<group>`s.

- `<summary>`: one line, starts lowercase, no trailing period
  (`command-line JSON processor`).
- `<needs-terminal/>`: command-line programs only.
- `license`: an attribute on `<group>` or `<implementation>`, holding an
  [SPDX identifier](https://spdx.org/licenses/) or expression.
- `stability`: hard-code `stable` unless the user publishes prereleases; then
  make it a `{stability}` placeholder (`testing` for prereleases).

## Arch tokens

`arch="OS-CPU"`; either side may be `*`. No `arch` means platform-independent
(scripts, JARs, .NET DLLs).

- OS: `Linux`, `Windows`, `Darwin` (macOS command-line programs), `MacOSX`
  (macOS GUI apps), `POSIX` (any Unix-like system).
- CPU: `x86_64`, `i486` (32-bit x86), `aarch64` (ARM64), `armv7l`.

The full list is in the
[specification](https://docs.0install.net/specifications/feed/#valid-architecture-names).

## `<archive>` and `<file>`

**`<archive href="..."/>`** unpacks to a directory tree.

- `extract`: the single top-level directory inside the archive to use as the
  root. 0template sets it when the archive has exactly one. Write it yourself
  when you want a subdirectory deeper down. Omit it when files sit at the
  archive's root.
- `type`: MIME type, needed only when the file extension doesn't give it away.
  The [specification](https://docs.0install.net/specifications/feed/#retrieval-methods)
  lists the supported formats.

Some formats only unpack on one OS: `.msi`, `.7z`, `.rar`, `.tar.zst` on
Windows, `.dmg` on macOS, `.deb`/`.rpm` on Linux. That limits them in two
ways:

- Users on other OSes can't use the implementation. Fine for a `.msi` in a
  `Windows-*` implementation; not fine for a `.7z` holding a Linux build.
  Prefer `.zip` and `.tar.gz`/`.tar.xz`, which unpack everywhere.
- 0template must run on an OS that can unpack every archive the template
  references. If none can (a `.dmg` and an `.msi` in one template), split the
  feed [per OS](#one-feed-per-os).

**`<file href="..." dest="name"/>`** stores one downloaded file as `name`.
Add `executable="true"` for a Unix binary or script.

`<command path>` is relative to the implementation root: the `extract`
directory if set, otherwise the archive root or the directory holding the
`<file>`.

## Groups

Attributes (`license`, `arch`, `stability`) and child elements (`<command>`,
`<requires>`, `<environment>`) on a `<group>` are inherited by its
implementations.

- Implementations with the same commands and dependencies: siblings in one
  group.
- Different command paths or dependencies per OS: one top-level group each,
  repeating the shared attributes. Don't nest groups; flat feeds merge more
  predictably.

## Several commands and entry points

One `<command name="..." path="..."/>` per executable, on the group.

- The main one is always `name="run"`.
- With both a CLI and a GUI main program: `run` is the CLI, `run-gui` the GUI.
- Others take the binary's own name. Users start them with
  `0install run --command NAME <feed>`.

`<entry-point>`s describe commands for desktop integration (menu entries,
aliases on `PATH`). They go after the metadata and before the groups:

```xml
<entry-point command="run" binary-name="myapp">
  <needs-terminal/>
  <name>MyApp</name>
  <summary>command-line interface</summary>
</entry-point>
<entry-point command="run-gui" binary-name="myapp-gui">
  <name>MyApp GUI</name>
  <summary>graphical interface</summary>
</entry-point>
```

- `binary-name`: the name for the launcher or alias. Needed whenever the
  command name differs from the executable name, so practically always for
  `run` and `run-gui`.
- `<needs-terminal/>`: on each entry point whose command is a console
  program. On Windows it decides whether a console window opens and whether a
  command-line alias is offered.
- `<name>`/`<summary>`: per entry point when there are several, so each gets
  its own label. A single-command feed can use a bare
  `<entry-point command="run" binary-name="myapp"/>`.

For fixed arguments (`<arg>`), a working directory (`<working-dir>`) and
file associations (`<capabilities>`), see the
[specification](https://docs.0install.net/specifications/feed/#commands) and
the [capabilities specification](https://docs.0install.net/specifications/capabilities/).

## Dependencies and bindings

`<requires interface="FEED-URL">` declares a dependency; 0install selects a
version of it too. A **binding** inside it says how the program finds the
dependency:

```xml
<requires interface="https://example.com/somelib.xml" version="2.0..!3">
  <environment name="PYTHONPATH" insert="lib"/>
</requires>
```

- `<environment name="VAR" insert="subdir"/>` adds `<dependency>/subdir` to
  `VAR`.
- `<executable-in-path name="tool"/>` puts a launcher for the dependency on
  `PATH`, for tools the program starts as a subprocess.
- `version="2.0..!3"` means at least 2.0 and before 3. Either end may be
  omitted (`17..`). A bare `2.0` matches exactly 2.0.

`<runner>` inside `<command>` is the special dependency that executes the
program; see [runtimes.md](runtimes.md). For everything else (`mode`,
`<executable-in-var>`, `<restricts>`, optional dependencies), see
[Dependencies](https://docs.0install.net/specifications/feed/#dependencies),
[Bindings](https://docs.0install.net/specifications/feed/#bindings) and
[Constraints](https://docs.0install.net/specifications/feed/#constraints).

## Icons

```xml
<icon href="https://example.com/myapp.png" type="image/png"/>
<icon href="https://example.com/myapp.ico" type="image/vnd.microsoft.icon"/>
<icon href="https://example.com/myapp.icns" type="image/x-icns"/>
```

| File | When |
| --- | --- |
| `.png` | Always, if the feed has any icon. Square, transparent, 128 to 512 px. |
| `.ico` | Feeds with Windows implementations (shortcuts, bootstrapper icon). |
| `.icns` | GUI apps with macOS implementations. |

Look for a logo in the project (`assets/`, `docs/`, the app's resources).
Prefer an `.ico`/`.icns` the project already ships. Otherwise convert with
ImageMagick; if `magick` isn't installed, replace it with
`0install run https://apps.0install.net/utils/imagemagick.xml`:

- SVG to PNG: `magick -background none logo.svg -resize 256x256 myapp.png`
- PNG to ICO: `magick myapp.png -define icon:auto-resize=256,48,32,16 myapp.ico`
- PNG to ICNS: **not** with `magick`, which writes a PNG under an `.icns`
  name. Use `iconutil` (macOS) or `png2icns` (Linux), or leave the `.icns` out.

Host the files next to the feed and commit them together with it. No logo
anywhere means no `<icon>` element.

## Letting 0template build the archive

When the feed is generated in the same build that produces the release, the
archive isn't online yet. 0template can
[pack it itself](https://docs.0install.net/tools/0template/):
give the implementation a `local-path` (the build output directory, relative
to the template) and the archive a relative `href`:

```xml
<implementation version="{version}" released="{released}" stability="stable" local-path="dist">
  <manifest-digest/>
  <archive href="myapp-{version}.tar.gz"/>
</implementation>
```

- The generated feed still has the relative `href`. Before publishing, the
  archive must be uploaded and the `href` replaced by its final absolute URL.
  In GitHub Actions the 0template action does that; see `0install-publish`.
- An archive built on Windows has no executable bits, so build Unix archives
  on Linux or macOS.

## One feed per OS

Split the feed when builds for different OSes are released on different
schedules, have different version numbers, or use archive formats no single
OS can unpack. Users then run a small hand-written top-level feed that has no
implementations, only metadata and one `<feed src="..." arch="Linux-*"/>` per
OS. The
[cross-platform guide](https://docs.0install.net/packaging/guides/cross-platform/)
shows the XML.

Each per-OS feed has its own template, repeating the top-level metadata. Mind
the two `<feed-for>` targets:

- In the **template**: the per-OS feed's own URL (`.../myapp-linux.xml`), so
  versions are merged into the right master feed.
- In the per-OS **master feed**, added once it exists: the top-level feed's
  URL (`.../myapp.xml`), whose interface its implementations belong to.

If all builds are released together and one OS can unpack them all, a single
feed with per-arch implementations is simpler.
