# clanker-conduct

Agent skills ([Agent Skills](https://agentskills.io)) for Claude, Cursor, Codex, and other compatible agents.

| Skill | Does |
| --- | --- |
| [`clanker-conduct`](skills/clanker-conduct/SKILL.md) | My preferences for how agents talk and code |
| [`tmux`](skills/tmux/SKILL.md) | Lets an agent open tabs and splits in its starting tmux session, run and read commands in them, and close what it opened |

## Install

```bash
# Choose from all skills
npx skills add mathewjhan/clanker-conduct

# Or install one
npx skills add mathewjhan/clanker-conduct --skill tmux
```

Or import the GitHub repo URL in your agent's skills / rules UI.

## Layout

```text
.
└── skills/
    └── <name>/
        ├── SKILL.md       # required — frontmatter + instructions
        ├── scripts/       # optional executable helpers
        ├── references/    # optional on-demand docs
        └── assets/        # optional templates / static files
```
