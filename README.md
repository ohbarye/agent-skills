# agent-skills

個人で使っている [Agent Skills](https://agentskills.io) の集まり。各 skill は `skills/<name>/SKILL.md`。

| skill | 内容 |
| --- | --- |
| [`cosense-memo`](skills/cosense-memo) | 会話や記事の要約を Cosense(旧 Scrapbox)形式のメモに変換する |
| [`magi-decision-support`](skills/magi-decision-support) | 3 つの視点(科学者・母・現実主義者)で意思決定を支援する |
| [`pr-lead-time`](skills/pr-lead-time) | PR のリードタイム(open → merge → deploy)を営業時間(JST)で計測し、週次・月次で集計する |
| [`review`](skills/review) | 5 つの専門サブエージェントで並列にコードレビューする |
| [`spotify-playlist-creator`](skills/spotify-playlist-creator) | ブラウザ自動化で Spotify のプレイリストを作成する |

## インストール

[`gh skill`](https://cli.github.com/manual/gh_skill) で入れられる。

```sh
gh skill install ohbarye/agent-skills pr-lead-time
gh skill install ohbarye/agent-skills --all
```

バージョンを固定する場合は `--pin v0.1.0`、または `<skill>@v0.1.0`。

## ライセンス

[MIT](LICENSE)
