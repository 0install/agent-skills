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

## Installation

With the [`skills` CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add 0install/agent-skills
```

Or copy `skills/0install` into your agent's skills directory, e.g. `.claude/skills/` in a project or `~/.claude/skills/` for Claude Code.
