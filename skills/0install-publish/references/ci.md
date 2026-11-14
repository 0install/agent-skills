# Publishing a feed from GitHub Actions on tag push

Pushing a `v*` tag builds the release, generates the per-version feed, merges
it into the master feed on the `gh-pages` branch, and re-signs it with a key
stored as a repository secret. The
[Zero Install GitHub Actions](https://github.com/0install/github-actions) wrap
0template and 0publish, so the workflow needs no 0install setup of its own.

Starting point: the manual flow in SKILL.md has been done at least once, so a
signed `myapp.xml`, its `<KEYID>.gpg` and any icons exist, and
`myapp.xml.template` is committed next to the source.

## Contents

- [What you set up, and what the user must do](#what-you-set-up-and-what-the-user-must-do)
- [1. The gh-pages branch](#1-the-gh-pages-branch)
- [2. The signing key secret](#2-the-signing-key-secret)
- [3. The workflow](#3-the-workflow)
- [Variations](#variations)
- [Action reference](#action-reference)

## What you set up, and what the user must do

You can write the workflow file and prepare the branch locally. These steps
are the user's, because they involve their private key or repository settings:

- exporting the GPG private key and storing it as the `GPG_KEY` secret;
- switching **Settings → Pages** to the `gh-pages` branch;
- pushing the branch, the workflow and the first tag (unless they tell you to).

Give them the exact commands and settings, and carry on with what doesn't
depend on them.

## 1. The gh-pages branch

The signed master feed, the public key and the icons live on an orphan
`gh-pages` branch; CI checks it out, updates it and pushes it back. Create it
from the files published by hand (here from `main`):

```bash
git checkout --orphan gh-pages
```

```bash
git rm -rf .
```

```bash
git checkout main -- myapp.xml "*.gpg" myapp.png
```

```bash
git commit -m "Initial gh-pages"
```

```bash
git checkout main
```

`git rm -rf .` empties the working tree of tracked files; run this only in a
clean checkout (`git status` shows nothing to commit), or use a separate
worktree: `git worktree add --orphan -b gh-pages ../myapp-pages`.

After the branch is pushed and Pages serves it (**Deploy from a branch /
gh-pages / root**), check that the feed URL still answers. The feed, key and
icon can then be deleted from `main`.

If the repository already has a `gh-pages` branch with a generated site, don't
replace it; see [Sharing gh-pages with a generated site](#sharing-gh-pages-with-a-generated-site).

## 2. The signing key secret

The user runs:

```bash
gpg --export-secret-keys --armor KEYID
```

and pastes the output into a new repository secret named `GPG_KEY`
(**Settings → Secrets and variables → Actions**). With the GitHub CLI, without
the key touching the clipboard or the shell history:

```bash
gpg --export-secret-keys --armor KEYID | gh secret set GPG_KEY
```

CI can't enter a passphrase. A key that has one fails with
`Inappropriate ioctl for device`. The usual answer is a dedicated signing key
without passphrase for this feed; that decision and the key handling are the
user's.

## 3. The workflow

`.github/workflows/publish.yml`:

```yaml
name: Publish
on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - uses: actions/checkout@v7
        with:
          ref: gh-pages
          path: gh-pages

      - name: Determine version
        id: version
        run: |
          echo "version=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"
          echo "date=$(date +%F)" >> "$GITHUB_OUTPUT"

      - name: Build release archives
        run: ./build.sh ${{ steps.version.outputs.version }}

      - name: Upload archives to GitHub Release
        uses: softprops/action-gh-release@v3
        with:
          files: |
            myapp-*.tar.gz
            myapp-*.zip

      - name: Generate per-version feed
        id: feed
        uses: 0install/github-actions/0template@v1
        with:
          template: myapp.xml.template
          version: ${{ steps.version.outputs.version }}
          args: released=${{ steps.version.outputs.date }}

      - name: Merge into master feed
        uses: 0install/github-actions/0publish@v1
        with:
          feed: gh-pages/myapp.xml
          add-from: ${{ steps.feed.outputs.feed }}
          gpg-key: ${{ secrets.GPG_KEY }}

      - name: Push gh-pages
        working-directory: gh-pages
        run: |
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git add -A
          git commit -m "Publish ${{ steps.version.outputs.version }}"
          git push
```

Adapt it to the project:

- **Build step.** Replace `./build.sh` with the project's real build. It must
  write each archive **next to the template, under the file name its
  `<archive href>` ends with**. 0template then digests that local copy rather
  than downloading, so the feed matches what gets uploaded.
- **Version.** The actions take the version as an input, so derive it however
  the project does (tag name, a file, GitVersion). It must be a valid 0install
  version: `1.2.3`, `1.2-rc1`; strip any `v`.
- **Order.** The archives must be on the release before users can fetch the
  new feed, so upload before the `gh-pages` push.
- **Runner.** `ubuntu-latest` can digest `.zip` and `.tar.*` archives for all
  platforms. A template with an `.msi` needs `windows-latest`, one with a
  `.dmg` needs `macos-latest`.
- **Placeholders.** `args` carries every `{key}` of the template other than
  `{version}`, as whitespace-separated `key=value` pairs. It is not run
  through a shell, which is why the date is computed in a step of its own.
  Remove `args` if the template has no `{released}`.

Before the first tag, check the workflow against the template: every
placeholder has a value, every `href` file name matches a build output, the
feed path under `gh-pages/` matches the feed URL.

## Variations

### Letting 0template build the archive

If the template uses `local-path` and a relative `href` (see `0install-feed`,
`references/feed-elements.md`), 0template packs the archive itself. Tell the
action where it will be published and it rewrites the `href`:

```yaml
      - name: Generate per-version feed
        id: feed
        uses: 0install/github-actions/0template@v1
        with:
          template: myapp.xml.template
          version: ${{ steps.version.outputs.version }}
          args: released=${{ steps.version.outputs.date }}
          github-release: true

      - name: Upload archives to GitHub Release
        uses: softprops/action-gh-release@v3
        with:
          files: ${{ steps.feed.outputs.archive }}
```

The upload step moves after the 0template step. `github-release: true` means
"the GitHub Release of the current tag"; `archive-url` names any other
location (a value ending in `/` is a directory).

### Sharing gh-pages with a generated site

Site publishers such as `peaceiris/actions-gh-pages` with `force_orphan: true`
replace the whole branch on every run, which would delete the feed. Keep the
`gh-pages` checkout and the 0publish step, drop the "Push gh-pages" step, and
copy the updated feed, the `.gpg` key and the icons into the directory the
site publisher uploads. The
[CI tutorial](https://docs.0install.net/packaging/tutorials/automate-ci/)
has the steps.

## Action reference

The workflow above uses the inputs that usually matter. For the rest (signing
a previously unsigned feed, choosing a key, setting stability, artifact
upload), read the `action.yml` of
[`0template`](https://github.com/0install/github-actions/blob/main/0template/action.yml)
and
[`0publish`](https://github.com/0install/github-actions/blob/main/0publish/action.yml).
