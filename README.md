# skill-agent-env-split

agent 产品双环境部署 SOP · main 分支 = prod · develop 分支 = staging · 域名 / 资源 / GHA workflow 三件套强制按环境区分。

## 三件套硬规则

1. **分支 → 环境映射** · main = prod / develop = staging / feature 分支不部署
2. **资源命名** · OSS bucket / FC function / RDS db / ACR tag / cert 全带 `-staging` 后缀
3. **域名命名** · staging 前缀放最前 · `staging.<service>.<product>.agentaily.com`

## GHA workflow 双轨

- `.github/workflows/deploy-staging.yml` · push develop trigger
- `.github/workflows/deploy-prod.yml` · push main trigger + e2e smoke gate

## 工作流

```
develop push → GHA staging deploy → 老板 UAT → merge main → GHA prod deploy
```

详见 `SKILL.md`。

## 关联

- `agent-product-layout` skill · 4 仓嵌套 + 6 域名表
- `fc-agent-deploy` skill · FC v3 + ACR 实施
- `aliyun-static-site` skill · OSS 静态站实施
- `~/.claude/memories/feedback_branch_env_split.md` · 老板 5-5 拍 · 5-9 重申

## 分支策略

- `main` = 主分支 (= 部署源 if 该仓有部署 · 否则 sync ~/.claude submodule)
- 临时分支: `feat/<name>` · `fix/<name>` · base 永远 `origin/main`
- PR target = `main` · CI pass + mergeable + 无 conflict 才合
- lead merge 不 auto · subagent 不自合
- subagent 创 worktree 干活前必读本 README

详见 `~/.claude/memories/feedback_lead_main_branch_only.md`
