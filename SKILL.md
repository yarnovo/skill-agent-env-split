---
name: agent-env-split
description: |
  agent 产品双环境部署 SOP · main 分支 = prod · develop 分支 = staging · 域名 / 资源 / GHA workflow 三件套强制按环境区分。
  TRIGGER when 老板/lead 起新 agent 产品需要双环境 / 配 GHA deploy.yml / 老板说 "分支区分环境 / staging 域名 / push develop 怎么 trigger / 双环境 / 环境怎么分"。
  DO NOT TRIGGER when 是单环境 demo (没 staging) · 是 maintainer 已熟双环境只问 1 个具体配置 (直接答) · 是改 cert / vault 凭证 (用 akong-cert / vault skill)。
argument-hint: ""
category: deploy
allowed-tools: Bash, Read, Edit, Write
---

# agent-env-split · 分支/环境/域名三件套规约 + GHA 集成

## 一句话

老板 5-5 拍 + 5-9 重申 · 每个 agent 产品仓硬规则:

- **main 分支 = prod** · 用户看到的 · 域名裸 (`<service>.<product>.agentaily.com`)
- **develop 分支 = staging** · 先验真 · 域名加 `staging.` 前缀 (`staging.<service>.<product>.agentaily.com`)
- **feature 分支** · PR 到 develop · 不部署 · 只跑 lint + test + build

push develop → GHA 自动 deploy staging · 老板/lead 跑 e2e + 浏览器 UAT · OK 才 merge main · GHA 自动 deploy prod。

## 老板原话

> "我们每个 agent 主分支是 prod · staging 分支就是 staging 环境 · 这个记住 · 我们用分支区分环境 · 域名也要区分" (5-9)

> "用 github actions 去做集成 · 可以考虑抽象为 skill" (5-9)

> "我们需要区分线上的 staging 和 prod 环境。" (5-5)

## 硬规则三件套

### 1. 分支 → 环境映射

| 分支 | 触发 | 环境 |
|---|---|---|
| `develop` | push | **staging**（先验真）|
| `main` | push（merge from develop）| **prod**（用户看到）|
| feature 分支 | PR 到 develop | 不部署 · 只 lint + test + build |

### 2. 资源命名

| 资源 | prod | staging |
|---|---|---|
| OSS bucket | `agentaily-<product>-<role>` | `agentaily-<product>-<role>-staging` |
| FC function | `<product>-<role>` | `<product>-<role>-staging` |
| RDS db | `<product>_<role>_prod` | `<product>_<role>_staging` |
| ACR image tag | `:latest` + `:<sha>` | `:staging-<sha>` |
| ACME cert | `<full-domain>` | `staging.<full-domain>` |

`<role>` ∈ {`agent`, `api`, `app`, `admin`}。

### 3. 域名命名 (5-7 老板拍 · 6 域名/agent)

每个 agent 产品 6 域名 = prod 3 + staging 3:

| 段 | prod | staging | 部署 |
|---|---|---|---|
| m | `m.<product>.agentaily.com` | `staging.m.<product>.agentaily.com` | OSS 静态站 (`<product>-app`) |
| api | `api.<product>.agentaily.com` | `staging.api.<product>.agentaily.com` | FC v3 (`<product>-api`) |
| agent | `agent.<product>.agentaily.com` | `staging.agent.<product>.agentaily.com` | FC v3 (`<product>-agent`) |

**重要** · staging 前缀**放在最前** · 不是中间段:

- ✅ 对: `staging.api.xiaoxi.agentaily.com`
- ❌ 错: `api.staging.xiaoxi.agentaily.com`
- ❌ 错: `staging-api.xiaoxi.agentaily.com`

可选第 4 段 admin (xiaoxi 已有): `admin.<product>.agentaily.com` (OSS · `<product>-admin`) · 加 6 → 8 域名。

## ENV 变量 (代码内逻辑区分)

每仓 deploy 时注入 env var:

```bash
ENV=prod   # main 分支 deploy 时
ENV=staging   # develop 分支 deploy 时
```

代码 default 行为按 ENV 真区分 (不再每处 if-else 写 hostname):

| 区分维度 | staging | prod |
|---|---|---|
| log 级别 | DEBUG | INFO |
| LLM key | DashScope test key (省钱) | DashScope prod key |
| db | sqlite local 或 RDS staging schema | RDS prod |
| feature flag | 全打开 (新功能 dogfood) | 按 release 滚 |
| sentry env tag | `staging` | `prod` |
| analytics endpoint | dev | prod |

代码：

```python
# api/main.py
import os
ENV = os.getenv("ENV", "staging")  # default staging · 防 prod 漏配
DEBUG = ENV == "staging"
LLM_API_KEY = os.getenv("DASHSCOPE_API_KEY")  # vault 注入 · staging/prod 不同
```

```typescript
// app/src/config.ts
export const ENV = import.meta.env.VITE_ENV;
export const IS_PROD = ENV === "prod";
export const API_BASE = IS_PROD
  ? "https://api.<product>.agentaily.com"
  : "https://staging.api.<product>.agentaily.com";
```

## GHA workflow 双轨

每个 capability 仓 `.github/workflows/`:

```
deploy-staging.yml    on: push to develop  → build + deploy staging
deploy-prod.yml       on: push to main     → build + deploy prod (含 e2e smoke gate)
```

### deploy-staging.yml 模板 (FC API/agent 仓)

```yaml
name: deploy-staging
on:
  push:
    branches: [develop]
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      ENV: staging
      ACR_REGISTRY: registry.cn-hangzhou.aliyuncs.com
      FC_FUNCTION_NAME: <product>-<role>-staging
      DOMAIN: staging.<role>.<product>.agentaily.com
    steps:
      - uses: actions/checkout@v4
      - name: docker build
        run: docker build -t $ACR_REGISTRY/yarnovo/<product>-<role>:staging-${GITHUB_SHA} .
      - name: ACR login + push
        run: |
          echo "${{ secrets.ACR_PASSWORD }}" | docker login $ACR_REGISTRY -u "${{ secrets.ACR_USERNAME }}" --password-stdin
          docker push $ACR_REGISTRY/yarnovo/<product>-<role>:staging-${GITHUB_SHA}
      - name: aliyun fc PUT
        run: |
          aliyun fc PUT /2023-03-30/functions/$FC_FUNCTION_NAME \
            --header "Content-Type=application/json;" \
            --body "{\"customContainerConfig\":{\"image\":\"$ACR_REGISTRY/yarnovo/<product>-<role>:staging-${GITHUB_SHA}\"}}"
      - name: e2e smoke
        run: curl -fsSL https://$DOMAIN/healthz
```

### deploy-prod.yml 模板

跟 staging 同结构 · 区别:

- `on: push: branches: [main]`
- `ENV: prod`
- `FC_FUNCTION_NAME: <product>-<role>` (无 `-staging` 后缀)
- `DOMAIN: <role>.<product>.agentaily.com`
- 镜像 tag `:latest` + `:${GITHUB_SHA}`
- e2e smoke gate · 失败 auto rollback (回上一个 tag)

### app 仓 (静态站 OSS) 模板

跟 FC 仓不同 · 走 `aliyun-static-site` skill:

```yaml
name: deploy-staging
on:
  push:
    branches: [develop]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - name: ossutil cp
        run: |
          ossutil cp -rf ./dist oss://agentaily-<product>-app-staging/ \
            --access-key-id "${{ secrets.OSS_AK }}" \
            --access-key-secret "${{ secrets.OSS_SK }}"
```

## 集成测试 gate

### staging deploy 完后

GHA 加 step 跑 e2e:

```yaml
- name: e2e playwright
  run: |
    npm install -D @playwright/test
    npx playwright install --with-deps chromium
    BASE_URL=https://staging.m.<product>.agentaily.com npx playwright test
```

通过才允许后续 PR merge main (老板手动 merge · GHA 不自动)。

### prod deploy 完后

```yaml
- name: prod smoke
  run: |
    curl -fsSL https://api.<product>.agentaily.com/healthz | grep -q "ok"
    curl -fsSL https://m.<product>.agentaily.com | grep -q "<title>"
```

失败 → GHA fail · 上报老板手动 rollback (上一个 ACR tag PUT 回去)。

## 启动 fail-fast

代码 boot 必 verify ENV + 关键 vault 注入:

```python
# api/main.py
import os, sys
ENV = os.getenv("ENV")
if ENV not in ("staging", "prod"):
    print(f"FATAL: ENV={ENV} not in (staging,prod)", file=sys.stderr)
    sys.exit(1)
for key in ("DASHSCOPE_API_KEY", "DB_URL"):
    if not os.getenv(key):
        print(f"FATAL: {key} missing", file=sys.stderr)
        sys.exit(1)
```

防生产空 env / 漏 vault 注入。

## 反例 (lead 别犯)

- ❌ prod 弹 debug alert (代码没看 ENV · 一律 alert)
- ❌ staging 不带 `staging.` 前缀 (域名跟 prod 撞)
- ❌ FC function name 不带 `-staging` 后缀 (deploy 互相覆盖)
- ❌ OSS bucket 不带 `-staging` 后缀 (静态站串)
- ❌ push develop 没 trigger (workflow 文件 `branches: [main]` 写错)
- ❌ feature 分支直接 push develop / main 跳过 PR
- ❌ hotfix 直接打 main (绕过 staging · 必走 develop · 除非 5 类必报老板拍)
- ❌ staging 跟 prod 共享 db / OSS / FC (数据互染)
- ❌ staging 用 prod LLM key (成本敏感时浪费)
- ❌ lockfile 锁老 commit hash 不重 update (deploy 跑老镜像 · 改了 GHA 不生效)

## 工作流 SOP

```
1. dev: git checkout develop && 改代码 && push
2. GHA 自动 deploy staging
3. 老板 / lead 跑 e2e + 浏览器手动 UAT staging.<service>.<product>.agentaily.com
4. OK → git checkout main && git merge develop && push
5. GHA 自动 deploy prod (e2e smoke gate · 失败 auto rollback)
```

## 例外 (5 类必报老板)

- 战略 hotfix 跨过 staging 直接 prod (事故 RCA / 安全事件) · 必报老板手动拍
- staging 数据需 prod 部分快照 (用户调研真实样本) · 必报老板拍数据脱敏方案

## 跟其他 skill 边界

| skill | 范围 |
|---|---|
| **agent-env-split** (本) | 双环境规约 + GHA 模板 (规约 + 集成) |
| **agent-product-layout** | 4 仓嵌套 + 6 域名表 (产品文件结构) |
| **fc-agent-deploy** | FC v3 + ACR 实施细节 (单仓单环境实操) |
| **aliyun-static-site** | OSS 静态站实施细节 (单仓单环境) |
| **akong-cert** | cert 签发 / 续期 (cert 周期) |

## 入口

- 老板 phone: "起新 agent 产品 X 的 GHA / 这仓加 staging" 等自然语言
- lead 起新仓 default 必跑本 skill 校 frontmatter 三件套

## 关联 memory

- `~/.claude/memories/feedback_branch_env_split.md` (5-5 拍 · 5-9 重申)
- `~/.claude/memories/feedback_agent_3_public_domains.md` (5-7 · 6 域名/agent)
- `~/.claude/memories/feedback_agent_product_file_layout.md` (5-9 · xiaoxi 范式)
