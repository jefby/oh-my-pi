# Roboomp — GitHub Triage/Fix Bot 架构

## 1. 概述

Roboomp（`python/robomp/`）是一个自托管的 GitHub triage + fix bot，把 **omp** 当作 headless agent 运行时：监听 allowlisted repo 的 webhook，在 per-issue git worktree 中驱动 `omp --mode rpc` 子进程，处理 issue/PR，并写回 GitHub。

- 位于 omp monorepo 内：源码 `python/robomp/`（Python + FastAPI），Web dashboard 是 Vite 构建的 SolidJS bundle。
- Docker 构建文件在仓库根目录 **`Dockerfile.robomp`**；它不构建 omp 本体，只 layer robomp 专属内容到 pi-base 之上。

## 2. 与 omp / pi-base 的关系

```
oh-my-pi/pi:dev (from /Dockerfile, `bun run pi:image`)
    ├─ python + bun + rustup launcher + pi-natives + omp_rpc wheel
    └─ /usr/local/bin/omp shim
        ▲
        │ FROM ${PI_BASE}
Dockerfile.robomp  → robomp:dev
```

- `pi-base`（`oh-my-pi/pi:dev`）提供完整 omp toolchain；`Dockerfile.robomp` 只加 Python 包 + dashboard bundle。
- 运行时把宿主机 omp checkout bind-mount 到 `/work/pi`（read-only），agent 配置从 `${HOME}/.omp/agent/models.container.yml` 和 `~/.agent/AGENTS.md`/`rules/` 经 staging 目录复制到容器内（见 §6）。
- 构建失效边界：改 robomp Python 只重建 runtime 层；改 pi 源码需先重新 `bun run pi:image`。

## 3. 双容器架构与信任边界

两个容器共享一个 Docker-managed volume (`/data`)，跨容器通信走 `internal: true` 网络：

| 容器 | 角色 | 持有凭据 |
|------|------|----------|
| **robomp**（orchestrator） | FastAPI + sqlite event queue + WorkerPool；运行 omp 子进程于 per-issue worktree；验证 webhook HMAC | HMAC key，**永不碰 PAT** |
| **gh-proxy**（sidecar） | `python -m robomp.proxy serve`；验 HMAC 后执行 REST + git push；只出网 `api.github.com` | `GITHUB_TOKEN`（PAT） |

- compose 用 per-service `environment:` allowlist，**刻意不用 `env_file:`** — PAT 只流入 gh-proxy。
- `robomp_internal` 网络无 ingress/egress；gh-proxy 无 host port mapping。
- 唯一例外：orchestrator 通过 `extra_hosts: llm-gateway.internal:host-gateway` 把宿主 LLM gateway（127.0.0.1:4000，LiteLLM-style）解析进容器。

## 4. 事件流

```
GitHub webhook (POST /webhook/github)
  → HMAC 验证 (GITHUB_WEBHOOK_SECRET)
  → github_events.route → sqlite `events` (dedup on X-GitHub-Delivery)
  → WorkerPool claim (BEGIN IMMEDIATE + per-(owner,repo,n) _inflight 集,
     max concurrency ROBOMP_MAX_CONCURRENCY, default 8)
  → sandbox.ensure_workspace → worktree farm/<8hex>/<slug> under /data/workspaces/
  → worker.run_task: spawn `omp --mode rpc` (cwd=worktree, persistent session_dir)
```

- **Issue 分类行为**（`issues.opened`）：
  - `bug` / `documentation` → 复现、在 fresh branch 修复、开 PR；body 固定含 `## Repro` / `## Cause` / `## Fix` / `## Verification` + `Fixes #N`。
  - `question` → 一条 comment（带 👎-to-keep-open prompt）；作者 `ROBOMP_QUESTION_AUTOCLOSE_HOURS`（默认 4h）内不点 👎 则 auto-close (`state_reason=completed`)；后续 comment 或外部 close 同步取消调度。
  - `enhancement` / `proposal` → 一条 comment，不开 PR。
  - `invalid` / `duplicate` → 简短 comment。
- **会话恢复**：issue/PR review 评论通过 `omp --mode rpc --continue` 恢复同一持久化 JSONL transcript；orchestrator 重启后 in-flight events 重入队、同样方式恢复。
- **Release sentinel**（默认关闭，见 §7）：`workflow_run` 事件驱动，在 reusable `main` worktree 中诊断失败的 release CI，原子 push 修复 commit + 既有 release tag，下一 verdict 继续同一会话，直到所有 run 与 GitHub Release 全绿。
- Agent 可用 omp 内置工具（`read`/`edit`/`bash`/`lsp`，scoped 到 worktree）+ `src/host_tools.py` 的 host tools — **GitHub 写操作的唯一面**；每次调用审计入 `tool_calls` 表（credential-redacted args/results）。

## 5. Dockerfile.robomp 结构

多阶段：

1. **web-builder**（`oven/bun:${BUN_VERSION}-slim`）：`bun install --filter robomp-web`（只 hydrate dashboard node_modules；`patches/*.patch` 必须在 root `bun install` 前存在，因 patchedDependencies）→ Vite build SolidJS bundle → `/work/python/robomp/web/dist/`。
2. **runtime**（`FROM ${PI_BASE}`）：拷入 `pyproject.toml` + `src/` + web-builder 的 `dist/`（落进 wheel，`static/**/*` 是 package-data）→ `pip install` fastapi/uvicorn/httpx/pydantic/pydantic-settings/dotenv/click → `--no-deps .` → 装 entrypoint。

- `VOLUME /data`；`EXPOSE 8080`（API，compose 映射到 host 6543）/ `8081`（gh-proxy）。
- `ENTRYPOINT [tini] → robomp-entrypoint`；默认 `CMD ["python", "-m", "robomp", "serve"]`，proxy 角色用 compose `command: python -m robomp.proxy serve`。

## 6. Entrypoint（`entrypoint.sh`）职责

- **slot 用户模型**：创建 `omp` group (GID 2000) + `omp-1..N` slot users/groups（N = `ROBOMP_MAX_CONCURRENCY`），每个 worktree 跑在独立 slot user 下，可被不同 slot 恢复中断工作。
- `/data/cache/{cargo,cargo-target,rustup,pi-natives}`：共享持久 build cache（per-worktree 共享一个 cargo target/toolchain；Bun install cache 刻意**不**共享 — chmod/chown 竞争）。权限统一为 group-writable (2770)。
- **agent-home staging**：宿主配置 read-only mount 到 `/srv/agent-home-stage` → entrypoint copy 成 root-owned world-readable files at `/srv/agent-home`；omp 子进程以 `HOME=/srv/agent-home` 运行，`~/.omp`/`~/.agent` 从那里解析，不暴露可变宿主 mount。
- `.omp/run`（daemon project registry）设 setgid + group-writable，任意 slot user 可建/进目录。
- `/data/robomp.sqlite` 及 -wal/-shm 强制 `root:root 0600`。

## 7. Release Sentinel（默认关闭）

开启条件：加 **Workflow runs** webhook event + PAT 需 **Actions: Read**。

- 只处理 subject 以 `ROBOMP_RELEASE_COMMIT_PREFIX`（默认 `chore: bump version to `）开头的 release commit。
- 每个 tag 持久化自己的 `releases` row + `.omp-session-<tag>` transcript；serialize under `<owner>/<repo>#release`。
- 可能直接 push 到 default branch 并移动既有 release tag — 因此默认 off，需显式开启。

## 8. 关键配置（`.env`）

| 变量 | 说明 | 默认 |
|------|------|------|
| `ROBOMP_REPO_ALLOWLIST` | 允许处理的 repos | 必填 |
| `GITHUB_WEBHOOK_SECRET` / `ROBOMP_GH_PROXY_HMAC_KEY` | webhook HMAC / gh-proxy 通道 HMAC（各自 `openssl rand -hex 32`） | 必填 |
| `ROBOMP_MODEL` | agent 模型（CSV 随机抽取） | `anthropic/claude-sonnet-4-6` |
| `ROBOMP_THINKING` | thinking level | `high` |
| `ROBOMP_MAX_CONCURRENCY` | worker slots / worktrees 并发 | 8 |
| `ROBOMP_TASK_TIMEOUT_SECONDS` | task timeout (+ hard grace) | 2400s (+60s) |
| `ROBOMP_QUESTION_AUTOCLOSE_HOURS` | question auto-close 窗口 | 4h |
| `ROBOMP_RELEASE_SENTINEL_ENABLED` | release sentinel | false |
| `ROBOMP_RATE_LIMIT_DEFAULT/CONTRIBUTOR` | per-contributor rate limit (window 3600s) | 3 / 10 |

容器内固定路径：`ROBOMP_WORKSPACE_ROOT=/data/workspaces`、`ROBOMP_SQLITE_PATH=/data/robomp.sqlite`、`ROBOMP_LOG_DIR=/data/logs`、`PI_ROOT=/work/pi`。

## 9. 构建与运行

```bash
bun run pi:image                          # 一次性 / pi 变更后: build oh-my-pi/pi:dev
bun run robomp:build && bun run robomp:up # compose up (默认 orchestrator)
curl -fsS http://localhost:8080/healthz
```

- 直跑 `docker build -f Dockerfile.robomp -t robomp:dev .`（context = pi root）。
- Webhook：Settings → Webhooks，payload URL `/webhook/github`，events = Issues, Issue comments, Pull requests, PR reviews/comments, Workflow runs（最后一条仅 release sentinel 需要）；GitHub `ping` 应在 1s 内返回 202。
- 公网入口只暴露 `/webhook/github`（Cloudflare/smee/ngrok 均可）；`/healthz`、`/events`、`/issues`、`/releases`、`/replay` 仅 localhost。
- In-process PAT 模式（host CLI/tests）：注释 `ROBOMP_GH_PROXY_URL`/`HMAC_KEY`、设 `GITHUB_TOKEN` — 两种模式互斥（`config.py` 拒绝双配）。

## 10. 关键文件

| 路径 | 作用 |
|------|------|
| `Dockerfile.robomp` | 多阶段构建（web-builder + runtime over pi-base） |
| `python/robomp/docker-compose.yml` | 双容器编排、网络隔离、env allowlist |
| `python/robomp/entrypoint.sh` | slot users、cache 权限、agent-home staging copy |
| `python/robomp/src/` | orchestrator（FastAPI + sqlite queue + WorkerPool）+ host_tools.py |
| `python/robomp/src/proxy` | gh-proxy（HMAC verify + REST/git push） |
| `python/robomp/web/` | SolidJS dashboard（Vite → dist，打进 wheel static/） |
