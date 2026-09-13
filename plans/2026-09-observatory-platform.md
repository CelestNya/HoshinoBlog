# Hoshino-Observatory 数据采集平台（规划存档）

> 状态：规划阶段，未实施。本文件记录已讨论确定的架构决定，供后续分批落地。
> 日期：2026-09-13

## 定位

独立于博客主题与内容仓的基础设施仓库，负责：多设备数据采集、统合分析（"大脑"）、
数据公网暂存与发布、管理员/访客视图分离。

## 仓库拓扑（三仓分工）

| 仓库 | 可见性 | 职责 |
|---|---|---|
| `CelestNya/HoshinoBlog` | public | 主题，贴上游，深度定制走 feat 分支 |
| `CelestNya/Shirone-Content` | public | 内容 + 数据快照（`data/` 实体） |
| `CelestNya/Hoshino-Observatory`（待建） | **private** | 采集平台：Worker 摄入 + D1/R2 + 分析大脑 + 设备代理 + 管理台 |

private 理由：含设备标识、token、基础设施代码。
注意：private 仓 GHA 免费额度 2000 分钟/月，"大脑"每日跑一次够用；小时级需提频策略。

## 数据流（一句话）

设备 → Worker 摄入（D1/R2）→ GHA 大脑分析（D1 + 快照）→ 公开端点（实时）+
内容仓快照（静态）→ 博客展示。

## Cloudflare 免费档分工

| 组件 | 免费额度 | 角色 |
|---|---|---|
| Workers | 10 万请求/天 | 摄入 API + 状态查询 + 数据发布点 |
| D1 (SQLite) | 5GB、10 万行写/天 | **暂存主库**：原始读数、聚合、心跳 |
| R2 | 10GB、零出口费 | 原始报文归档 |
| Workers Cron | 免费 | 轻聚合（10ms CPU 内） |

- KV 免费档 1000 写/天，心跳不能走 KV，必须走 D1。
- Workers 免费 CPU 10ms，重分析必须在 GHA（Python 经 D1 HTTP API 读写）。

## 平台仓结构（拟定）

```
Hoshino-Observatory/
├── worker/                  # Cloudflare Worker（TypeScript + Hono）
│   ├── src/index.ts         # 路由：ingest / presence / public / admin
│   ├── src/schema.sql       # D1 迁移
│   └── wrangler.toml        # D1 + R2 + Cron 绑定
├── brain/                   # GHA Python 分析任务
│   ├── aggregate.py         # 清洗→窗口聚合→写回 D1 aggregates
│   └── snapshot.py          # 产出快照 JSON → PR 到内容仓 data/
├── agents/posix/            # 试点采集代理（shell/Node，cron 驱动）
├── admin/                   # 管理台 SPA（后置）
└── .github/workflows/       # deploy.yml（wrangler-action）、brain.yml（定时）
```

## D1 schema（草案）

- `devices(id, name, kind, token_hash, created_at)` — token 只存哈希
- `readings(id, device_id, metric, value, ts)` — 索引 (device_id, ts)
- `presence(device_id, last_seen, meta)` — 心跳 upsert；在线 = last_seen > 90s
- `aggregates(name, window, payload_json, computed_at)` — 大脑产出

## Worker 端点

- `POST /api/v1/ingest` — 设备 Bearer token，批量指标 + 心跳；D1 + R2 归档
- `GET /api/v1/presence` — 公开，设备在线状态（博客小组件轮询）
- `GET /api/v1/public/:stream` — 公开聚合数据（Cache API）
- `/api/admin/*` — 校验 Cloudflare Access JWT（`Cf-Access-Jwt-Assertion`）

## 管理员/访客分离

- 访客：博客内展示聚合数据（实时小组件拉 Worker 公开端点；历史页用内容仓快照静态渲染）。
- 管理员：Worker 托管的独立 SPA，Cloudflare Access（Zero Trust 免费版）做门禁，
  零认证代码；Worker 校验 Access JWT 区分敏感数据。
- 备选：上游 `feat/go-backend-foundation` 已有完整会话+角色体系
  （Argon2id、member/editor/admin、SSR 条件渲染），若博客将来动态化可复用。

## 博客侧集成点（在 feat/hoshino-custom 落地时）

1. 新 config 域 `observatoryConfig`（`src/config/` + `src/types/`，`pnpm schemas`）。
2. devices 页加 presence 小组件，轮询 Worker；`enable=false` 时零 DOM 零请求。
3. 指标仪表盘：新页面/organisms 组件，数据来自内容仓快照。
4. 契约守卫：packaging contract、manifests、i18n（至少 zh_CN/en）、e2e 测试。

## 试点范围（v0）

1. 建仓 + Worker 部署（需 CF 账号 wrangler login）
2. D1 schema + ingest + presence 端点
3. 试点代理：一台服务器/NAS 每 60s 上报 cpu/mem/disk/net + 心跳（~1400 请求/天）
4. GHA 大脑：每日聚合 + 快照 PR 到内容仓
5. 博客：devices 页在线状态组件 + 简单仪表盘页
6. 管理台：Access 保护的只读 SPA 骨架（可后置 v1）

## 分批落地原则

整个系统分多次迭代，每次一个独立可验证的小批次；本分支（feat/hoshino-custom）
持续记录规划与进展，实施工作按批次另开 feat 分支。
