# Contributing to agent-env-split

akong 团队 skill 仓 · agent-env-split

## Branching Strategy

- `main` = 主分支 (= 部署源 if 该仓有部署 · 否则 sync ~/.claude submodule)
- 临时分支: `feat/<name>` · `fix/<name>` · base 永远 `origin/main`
- PR target = `main` · CI pass + mergeable + 无 conflict 才合
- lead merge 不 auto · subagent 不自合
- subagent 创 worktree 干活前必读 README + 本 CONTRIBUTING.md

详见 `~/.claude/memories/feedback_lead_main_branch_only.md`
