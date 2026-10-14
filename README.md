# agent-skills

A collection of personal [Agent Skills](https://agentskills.io). Each skill lives in `skills/<name>/SKILL.md`.

| Skill | Description |
| --- | --- |
| [`cosense-memo`](skills/cosense-memo) | Turns a conversation or an article summary into a note in Cosense (formerly Scrapbox) format. |
| [`magi-decision-support`](skills/magi-decision-support) | Multi-perspective decision support with three agents (Scientist, Mother, Realist). |
| [`pr-lead-time`](skills/pr-lead-time) | Measures PR lead time (open → merge → deploy) in business hours (JST) and aggregates it weekly or monthly. |
| [`review`](skills/review) | Code review with five specialized subagents running in parallel (quality, performance, tests, docs, security). |
| [`spotify-playlist-creator`](skills/spotify-playlist-creator) | Creates Spotify playlists by driving the desktop app with browser automation. |

## Requirements

Some skills depend on tools or agents that are not part of this repository.

| Skill | Needs |
| --- | --- |
| `pr-lead-time` | Ruby (stdlib only) and an authenticated `gh` CLI |
| `spotify-playlist-creator` | [`agent-browser`](https://github.com/vercel-labs/agent-browser) and the Spotify desktop app |
| `review` | Subagents `code-quality-reviewer`, `performance-reviewer`, `test-coverage-reviewer`, `documentation-accuracy-reviewer`, `security-code-reviewer` (see [`claude/agents`](https://github.com/ohbarye/dotfiles/tree/main/claude/agents) in my dotfiles) |
| `magi-decision-support` | Subagents `melchior`, `balthasar`, `casper` (same place) |

## Install

With [`gh skill`](https://cli.github.com/manual/gh_skill):

```sh
gh skill install ohbarye/agent-skills pr-lead-time      # one skill
gh skill install ohbarye/agent-skills --all             # everything
gh skill install ohbarye/agent-skills pr-lead-time --pin v0.1.0
```

## Development

`pr-lead-time` ships with tests:

```sh
cd skills/pr-lead-time
for f in test/*_test.rb; do ruby -Ilib -Itest "$f"; done
```

Validate before publishing:

```sh
gh skill publish --dry-run
```

## License

[MIT](LICENSE)
