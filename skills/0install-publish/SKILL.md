---
name: 0install-publish
description: >-
  Publish a Zero Install (0install) feed so anyone can run the app with
  `0install run <url>`: build the master feed from per-version feeds with
  0publish, sign it with GPG, host it on GitHub Pages (or any static web
  host), add new versions, and automate all of it with GitHub Actions on tag
  push. Use when the user wants to publish or release a 0install feed, sign a
  feed, add a new version to a published feed, set up CI that publishes the
  feed on release, or create a bootstrapper/installer for the app. For writing
  the feed template itself, see the `0install-feed` skill.
---

# Publishing a Zero Install feed

A feed becomes usable by others once the **master feed** (the one listing all
versions) is **signed** and served from its **permanent URL**:

```
myapp-1.2.xml ──(0publish --add-from)──▶ myapp.xml ──(GPG signature)──▶ https://OWNER.github.io/myapp/myapp.xml
 per-version feed                        master feed                     + <KEYID>.gpg next to it
```

This skill starts from a template (`myapp.xml.template`) and a per-version
feed generated from it (`myapp-1.2.xml`). If those don't exist yet, create
them with the `0install-feed` skill first.

Two ways to publish:

- **By hand** (below): right for the first release and for trying things out.
- **CI on tag push**: [references/ci.md](references/ci.md). Set this up once
  the manual flow has worked, or straight away if the user asks for
  automation.

This covers one feed published from its own repository. Several feeds sharing
one signing key and catalog call for a
[0repo](https://docs.0install.net/packaging/tutorials/multi-feed/) feed
repository instead, which is outside this skill.

## Conventions

- `0install` means a working 0install command; see the `0install` skill if it
  isn't on `PATH`. Tools run through it:
  `0install run https://apps.0install.net/0install/0publish.xml ...`, written
  `0publish ...` below for brevity. Likewise `0template`, `feedlint` and
  `0bootstrap` (`https://apps.0install.net/0install/<tool>.xml`).
- `OWNER`, `myapp` and version numbers are placeholders.

## Guardrails

Publishing is outward-facing and hard to take back: users' machines cache the
feed and trust the key.

- **The feed URL is permanent.** Other feeds and every user's cache refer to
  it. Confirm it with the user before the first signed feed is pushed.
- **Signing uses the user's private key.** Ask before the first signing. Never
  create a GPG key, export a private key, remove a passphrase, or enter a
  passphrase on the user's behalf: give them the command and let them run it.
- **Don't push** the signed feed (or tags that trigger a publishing workflow)
  unless the user asked for it in this conversation.
- **Never change a published implementation** (its `id`, digest, archives),
  and never replace a released archive with a different file under the same
  URL. Users who have the old digest get "Incorrect manifest" errors. To
  withdraw a broken version, set its `stability` to `buggy`; to fix it,
  release a new version.
- Private keys and tokens never go into the repository, the feed, or chat.

## Step 1: Check the prerequisites

- **A GPG secret key**: `gpg --list-secret-keys --keyid-format long`. If there
  is none, the user creates one with `gpg --full-generate-key` (defaults are
  fine; name and email are shown to users when they are asked to trust the
  key). With several keys, ask which to use and pass it as `-k KEYID`.
- **A place to host static files over HTTPS.** For GitHub Pages: the repo's
  **Settings → Pages**, source "Deploy from a branch". Feed and archives may
  live in different places; the feed URL is what must stay stable.
- **Release archives online** at the URLs the template names, e.g. attached
  to a GitHub Release.
- **A per-version feed that passes verification** (`0install-feed`, step 4),
  with a `<feed-for>` naming the final feed URL.

## Step 2: Create and sign the master feed

In the directory that is served as the feed URL's parent:

```bash
0publish myapp.xml --add-from=myapp-1.2.xml --xmlsign
```

Because `myapp.xml` doesn't exist yet, 0publish turns the per-version feed
into the master feed: the `<feed-for>` URL becomes the `uri` attribute of
`<interface>`, and the `<feed-for>` element is dropped. `--xmlsign` appends
the signature as an XML comment at the end of the file and exports the public
key next to the feed as `<KEYID>.gpg`.

GPG asks for the passphrase through its own prompt (pinentry). If the command
fails or hangs because no prompt can be shown, hand the command to the user
to run in their own terminal. Don't work around it.

Check the result:

- `myapp.xml` starts with `<interface uri="https://.../myapp.xml" ...>` and
  ends with a `<!-- Base64 Signature ... -->` comment.
- A `<KEYID>.gpg` file sits next to it.
- `0install select ./myapp.xml` still resolves.

## Step 3: Host it

The feed, the key file and any icons the feed references go to the web host
together. With GitHub Pages serving the branch root:

```bash
git add myapp.xml *.gpg myapp.png
```

```bash
git commit -m "Publish 1.2"
```

Push only with the user's go-ahead. Not for the repo: the per-version feeds
(`myapp-*.xml`) and downloaded archives; `.gitignore` them.

Once the host has updated (GitHub Pages takes a minute or two), verify the
published feed, not the local file:

```bash
feedlint https://OWNER.github.io/myapp/myapp.xml
```

```bash
0install run --refresh https://OWNER.github.io/myapp/myapp.xml --version
```

feedlint checks the signature, that the key can be downloaded, and that the
icon, homepage and archive URLs work. The first run from a URL asks whether to
trust the signing key for that domain; on the user's own machine that is their
call to confirm.

## Adding a version

```bash
0template myapp.xml.template version=1.3 released=2026-03-01
```

```bash
0publish myapp.xml --add-from=myapp-1.3.xml
```

`--add-from` copies the new `<implementation>`s into the matching `<group>` of
the master feed and re-signs it with the key it was already signed with (no
`--xmlsign` needed). Verify the per-version feed before merging it, then
commit and push `myapp.xml` as in step 3.

Only implementations are copied. Changes to the template's metadata, commands
shared at group level or `<entry-point>`s don't reach the master feed this
way; make them in the master feed too (next section).

## Editing a published feed

The signature covers the whole file, so any edit invalidates it. After
changing `myapp.xml` in an editor, re-sign:

```bash
0publish myapp.xml --xmlsign
```

Then check with `0install select ./myapp.xml` before pushing. Note that
0publish may reformat the XML and drop comments.

Useful single-purpose edits (each re-signs a signed feed):

| Goal | Command |
| --- | --- |
| Mark a version as broken | `0publish myapp.xml --select-version=1.2 --set-stability=buggy` |
| Promote the newest testing version | `0publish myapp.xml --stable` |
| Sign with a different key | `0publish myapp.xml -k NEWKEYID --xmlsign` (publish the new `.gpg` too; users are asked to trust it) |

## Optional: an installer for users without 0install

[0bootstrap](https://docs.0install.net/tools/0bootstrap/) wraps a published
feed in a small executable that fetches 0install if needed and then runs the
app. It needs the feed to be online. On Windows:

```powershell
0bootstrap https://OWNER.github.io/myapp/myapp.xml
```

This writes `run MyApp.exe`, named after the feed's `<name>` and using its
icon (`--output=` to choose the name).

- `--integrate-args="--add-standard"` makes it add the app to the Start menu
  and register its file types instead of just running it.
- For an offline installer, bundle the app:
  `0install export --include-zero-install <feed-url> export`, then add
  `--content=export\content` to the 0bootstrap call.

Attach the result to the release or host it next to the feed. On Linux and
macOS, 0bootstrap produces `.deb`/`.rpm` packages; see its documentation.

## Troubleshooting

`gpg: signing failed: secret key not available`
: The feed was last signed with a key that isn't on this machine. Find out
which key the user wants and pass `-k KEYID`.

`Incorrect manifest -- archive is corrupted` / `An implementation's manifest hash does not match the expected value`
: The file at an archive URL is not the one the digest was computed from.
Either the upload replaced an existing archive, or 0template used a stale
local copy with the same file name. Never overwrite a released archive;
release a new version.

The key-trust prompt shows no name, or 0install can't find the key
: The `<KEYID>.gpg` file isn't in the same directory as the feed on the web
host.

Users don't see the new version
: Their cached copy of the feed is still fresh. `0install run --refresh <url>`
or `0install update <url>` fetches it now.

## Further reading

- Tutorials: [Publishing your own app](https://docs.0install.net/packaging/tutorials/publish-app/),
  [Automating a single feed with CI](https://docs.0install.net/packaging/tutorials/automate-ci/).
- [0publish reference](https://docs.0install.net/tools/0publish/).
