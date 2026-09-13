# 本地开发速查（HoshinoBlog / Shirone 定制线）

> 适用范围：`feat/hoshino-custom` 及后续定制分支。上游原貌见 `AGENTS.md` 与 `docs/`。

## 起步

```bash
pnpm install
pnpm dev        # → http://localhost:4321
```

- 无 `shirone.content.json` 时自动使用内置示例内容（`src/content/`），sync 静默跳过，无需任何 .env。
- dev 预步骤（图标生成、moment 缩略图）全部离线；首次启动会下载 fontsource 字体并缓存。
- 项目锁定 pnpm 9.14.4（`packageManager` 字段）；本机 pnpm 版本不一致时用 `corepack` 切换。
- Node ≥ 22.12（CI 用 22/24 双版本验证）。

## 检查闭环（改完代码跑什么）

| 目的 | 命令 |
|---|---|
| Astro 类型检查（提交前必须零错误） | `pnpm check` |
| Lint/格式只读校验 | `pnpm exec biome ci ./src` |
| 单个 e2e 测试 | `pnpm test tests/site/<spec>.spec.ts` |
| 全部 node 单测 | `node --test "tests/**/*.test.mjs"` |
| 设计 token 守卫 | `pnpm design:lint` |
| manifest 守卫（atoms/markdown/skills） | `pnpm check:manifest` |

- Playwright 复用系统 Chrome（`channel: "chrome"`），自动起 4321 dev server，已有实例则复用。
- 视觉回归快照在 `tests/site/visual.spec.ts`，改 UI 后需有意识地更新。

## 构建与产物

```bash
pnpm build      # sync → icons → thumbnails → 字体子集 → astro build → pagefind → 字体校验
pnpm preview    # 本地预览 dist/
```

- 输出为**纯静态** `dist/`（无 adapter），任何静态托管可用。
- 字体子集是 build 专属步骤（dev 不需要）；meting 音乐源的字体收集失败不致命。

## 内容接入（以后接 Shirone-Content 时）

- 方式 A：根目录 `shirone.content.json`（模板见 `shirone.content.example.json`），声明内容仓 git url + 挂载表。
- 方式 B：环境变量 `CONTENT_DIR` 指向本地内容仓目录（开发期最方便）。
- `pnpm content:watch` 增量热更新；`pnpm content:validate` 干跑校验。

## 改动守则（定制线纪律）

1. feat 分支只从 `dev` 切出，完成后合回 `dev`，`dev` 验证后才合 `main`（main 有分支保护）。
2. 上游同步：`git fetch upstream` 后在 `dev` 上 merge `upstream/main`，改造尽量做"加法"少碰上游行。
3. 新配置域：`src/config/<domain>Config.ts` + `src/types/` 对应类型 +（需内容仓覆盖时）登记 `scripts/content/config-domains.mjs`，然后 `pnpm schemas`。
4. 双模式契约：源码改动需同时保持"源码模式"和 shirones npm 包模式可用（`docs/packaging-contract.md`）。
5. i18n：新增界面文案至少补 zh_CN 与 en。

## 部署（待定）

产物纯静态，构建命令 `pnpm build`、输出目录 `dist`；CF Pages / Vercel / Netlify / GH Pages
均零配置可接。上游 `deploy.yml.example` 与内容仓 `trigger-build.yml.example` 是现成模板，
选定平台后再启用。
