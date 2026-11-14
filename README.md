# Zero Install agent skills

Agent skills that teach AI coding agents how to work with [Zero Install](https://0install.net/).

## Skills

### `0install`

Lets an agent run command-line tools it needs for a task (e.g. Node.js, jq, 7-Zip) with `0install run` instead of installing them. Tools are downloaded into Zero Install's per-user cache, so your `PATH`, package manager and system directories stay untouched.

The skill teaches the agent to:

- find a tool's feed with `0install search` or `0install catalog search`
- run it, optionally pinning a version or choosing a specific command from the feed
- stop and ask you before trusting an unknown signing key or installing a distribution package
- add a persistent command to your `PATH` only if you explicitly ask for one

Zero Install does not need to be installed beforehand. The skill bundles the [bootstrap scripts](https://get.0install.net/) `0install.sh` and `0install.ps1`, which download Zero Install into the per-user cache on first use. It works on Windows, Linux and macOS.

### `0install-feed`

Lets an agent package a program for Zero Install: write a feed template, generate the feed for a release with [0template](https://docs.0install.net/tools/0template/), then lint and test-run it locally.

The skill teaches the agent to:

- gather what a feed needs (metadata, download URLs, platforms, runtime) from the project
- write the template, with recipes for .NET, Java, Python and native Windows, Linux and macOS packages as well as libraries
- verify the generated feed with feedlint, `0install select` and `0install run`
- write a local feed to run a source checkout without a release

### `0install-publish`

Lets an agent publish a feed so that anyone can run the app with `0install run <url>`.

The skill teaches the agent to:

- build and sign the master feed with [0publish](https://docs.0install.net/tools/0publish/) and host it on GitHub Pages or another static web host
- add new versions and edit a published feed
- set up a GitHub Actions workflow that publishes a new version on every tag push
- wrap the app in an installer with [0bootstrap](https://docs.0install.net/tools/0bootstrap/)
- leave key handling to you, and ask before signing or pushing anything

## Installation

With the [`skills` CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add 0install/agent-skills
```

Or copy the folders under `skills/` into your agent's skills directory, e.g. `.claude/skills/` in a project or `~/.claude/skills/` for Claude Code.
