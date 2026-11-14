# Recipes by runtime and package format

Each section gives the part of the template that differs from the skeleton in
SKILL.md, the traps, and a link to the full packaging guide. Read the
linked guide when the case at hand goes beyond what is shown here.

A `<runner>` names the feed of the interpreter or runtime that executes the
`<command path>`. 0install selects and downloads it like any dependency, so
users need no system-wide install. Constrain its version only as tightly as
the program really requires.

## .NET (Core, 5+)

Guide: <https://docs.0install.net/packaging/guides/dotnet/>

Runtime feeds, versioned like .NET itself, all under
`https://apps.0install.net/dotnet/`: `runtime.xml` (console and other
non-UI apps), `aspnetcore-runtime.xml`, `windowsdesktop-runtime.xml` (WPF,
Windows Forms).

```xml
<command name="run" path="MyApp.dll">
  <runner interface="https://apps.0install.net/dotnet/runtime.xml" version="8.0..!9.0"/>
</command>
```

- `path` is the `.dll`, not the `.exe` apphost.
- Match the version range to the target framework; a `net8.0` app doesn't run
  on the 9.0 runtime unless it opts into roll-forward.
- A framework-dependent build is one implementation without `arch`.
  Self-contained builds need no runner: treat them as native binaries, one
  implementation per `arch`.

## .NET Framework

Guide: <https://docs.0install.net/packaging/guides/dotnet-framework/>

```xml
<command name="run" path="MyApp.exe">
  <runner interface="https://apps.0install.net/dotnet/clr.xml" version="4.7.2.."/>
</command>
```

- `clr.xml` selects the .NET Framework on Windows and Mono elsewhere. Versions
  are framework versions (`4.5`, `4.8`).
- Set `arch="Windows-*"` if the app is Windows-only (WPF, Win32 P/Invoke);
  leave `arch` off if it also runs on Mono.
- Library DLLs from other feeds need `clr-monopath.xml` as the runner; see the
  guide.

## Java

Guide: <https://docs.0install.net/packaging/guides/java/>

A single self-contained JAR, downloaded as a `<file>`:

```xml
<command name="run" path="myapp.jar">
  <runner interface="https://apps.0install.net/java/jre.xml" version="17..">
    <arg>-jar</arg>
  </runner>
</command>
<implementation version="{version}" released="{released}" stability="stable">
  <manifest-digest/>
  <file href="https://example.com/myapp-{version}.jar" dest="myapp.jar"/>
</implementation>
```

- Versions are feature releases (`8`, `17`, `21`). Too low a floor shows up at
  launch as `UnsupportedClassVersionError` (class file 52 = Java 8, 55 = 11,
  61 = 17, 65 = 21).
- `java -jar` ignores `CLASSPATH`. An app with a `lib/` folder of JARs, or
  with library JARs from other feeds, needs
  `https://apps.0install.net/java/jar-launcher.xml` as the runner instead; the
  guide has the pattern.
- Don't use the `bin/myapp` / `bin/myapp.bat` start scripts many Java
  distributions ship as the command: they look for a system Java. Point at the
  JAR.

## Python

Guide: <https://docs.0install.net/packaging/guides/python/>

```xml
<command name="run" path="mytool.py">
  <runner interface="https://apps.0install.net/python/python.xml" version="3.10.."/>
</command>
```

- GUI apps: add `command="run-gui"` to the runner (`pythonw` on Windows, no
  console window) and omit `<needs-terminal/>`.
- PyPI dependencies: bundle them in the archive
  (`pip install --target=lib ...` at release time) and add
  `<environment name="PYTHONPATH" insert="lib"/>` to the group.

## Other interpreters

Same pattern, different runner:

| Runtime | Feed |
| --- | --- |
| Node.js | `https://apps.0install.net/javascript/node.xml` |
| Perl | `https://apps.0install.net/perl/perl.xml` |
| Ruby | `https://apps.0install.net/ruby/ruby.xml` |
| PowerShell | `https://apps.0install.net/powershell/powershell.xml` |

Check that the feed has a version for the platforms you target before relying
on it: `0install select <feed>`.

## Native Windows

Guide: <https://docs.0install.net/packaging/guides/windows-native/>

- CPU tokens: `x86_64` (x64), `i486` (x86), `aarch64` (ARM64).
- **Console or GUI.** A console `.exe` needs `<needs-terminal/>`, at the top
  of the feed and on its `<entry-point>`. A GUI `.exe` must not have it. With
  both variants: `run` is the console one, `run-gui` the GUI one.
- **MSI** is unpacked without being installed (`type="application/x-msi"`).
  `extract` is the path inside the MSI's directory table, e.g.
  `SourceDir/MyApp`. It differs per package, so generate once without it and
  inspect the result. 0template can only process an `.msi` on Windows.
- Setup programs that can't be unpacked (`setup.exe`) can't be referenced. The
  app has to be offered as an archive.
- The app must run from an arbitrary read-only directory: no writing next to
  its executable, no reliance on registry entries an installer made.
- Give the feed an `.ico` in addition to the PNG.

## Native Linux

Guide: <https://docs.0install.net/packaging/guides/linux-native/>

- A single binary (typical for Go and Rust tools) is a `<file>`:
  `<file href="..." dest="myapp" executable="true"/>`.
- **`.deb` / `.rpm`** are unpacked without being installed. Paths lose their
  leading slash: `/usr/bin/myapp` becomes `path="usr/bin/myapp"`. This only
  works if the program is relocatable, i.e. doesn't hard-code
  `/usr/share/...`. With several packages to choose from, take the one for the
  oldest distribution.
- **AppImage** can't be unpacked by 0install. Use a tarball if upstream has
  one; otherwise the AppImage must be re-archived and hosted (see the guide).

## Native macOS

Guide: <https://docs.0install.net/packaging/guides/macos-native/>

- `arch`: `Darwin-*` for command-line tools, `MacOSX-*` for GUI apps; CPU
  `x86_64` (Intel), `aarch64` (Apple Silicon), `*` for universal binaries.
- **`.app` bundles** are directories. Point the command at the executable
  inside: `path="MyApp.app/Contents/MacOS/MyApp"`.
- **`.dmg`**: `type="application/x-apple-diskimage"`. 0template can only
  process it on macOS.
- GUI apps: add an `.icns` icon in addition to the PNG.

## Libraries

Guide: <https://docs.0install.net/packaging/guides/libraries/>

A library feed is a feed without a `run` command; native libraries get one
implementation per `arch`. Consumers pull it in with `<requires>` plus a
binding that depends on how the library is loaded (`LD_LIBRARY_PATH`,
`PYTHONPATH`, `CLASSPATH`, ...); the guide lists the binding per ecosystem.

`0install select` works on a library feed, but `0install run` rightly fails,
so verify it through a consumer.
