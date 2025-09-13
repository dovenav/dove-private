# dove-private

基于 [dove](https://github.com/dovenav/dove) 的个人（或组织）导航站点私有配置仓库。通过 GitHub Actions 自动构建并发布到 GitHub Pages。

本仓库只存放你的站点配置（`dove.yaml` 等），实际构建时会从 GitHub 拉取 [dovenav/dove](https://github.com/dovenav/dove) 源码，在 CI 里执行 `cargo run -- build` 产出静态文件并发布。

## 快速开始

- Fork 或新建一个仓库，将你的 `dove.yaml` 放在仓库根目录。
- 选择部署方式：
  - GitHub Pages：在 Settings → Pages 中将 Source 设为 “GitHub Actions”。
  - Cloudflare Workers：在仓库设置添加 Secrets `CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`。
- 推送到 `main`，工作流会自动构建并发布。

建议：若为 Project Pages（`https://<user>.github.io/<repo>/`），在 `dove.yaml` 设置 `site.base_path: <repo>`；User/Org Pages 通常无需设置。

## 工作流总览

- GitHub Pages 发布：`.github/workflows/deploy.yml`
  - 触发：`push` 到 `main`、手动 `workflow_dispatch`
  - 产物：`dove/dist` → GitHub Pages
- Cloudflare Workers 发布：`.github/workflows/deploy-worker.yml`
  - 触发：`push` 到 `main`、手动 `workflow_dispatch`
  - 产物：`dove/dist` → 通过 Wrangler 发布为静态资源 Worker（配置见 `wrangler.toml`）

## 实现方式概览

- 仓库内容：
  - `dove.yaml` 站点配置（可根据需要新增其他资源，如自定义静态目录或主题文件）。
  - `.github/workflows/deploy.yml` GitHub Pages 自动发布工作流。
- 工作流关键步骤：
  1) Checkout 当前仓库（只含配置）。
  2) Checkout [dovenav/dove](https://github.com/dovenav/dove) 到子目录 `dove/`。
  3) 将 `dove.yaml` 拷贝到 `dove/` 根目录。
  4) 安装 Rust 工具链，执行 `cargo run -- build` 生成 `dove/dist/`。
  5) 上传 `dove/dist` 为 Pages 工件并发布到 GitHub Pages。

## 一次性准备（仓库 & Pages 设置）

1) 新建一个 GitHub 仓库（公开或私有均可，GitHub Pages 在私有仓库需要付费计划）。
2) 在仓库 Settings → Pages：
   - Build and deployment → Source 选择 “GitHub Actions”。
3) 将你本地准备好的 `dove.yaml` 提交到该仓库的默认分支（示例为 `main`）。
4) 如需项目页（Project Pages）路径正确，建议在 `dove.yaml` 中设置：
   - `site.base_path: <你的仓库名>`（例如 `site.base_path: privatedove`）。
   - 如果使用 User/Org Pages（`<username>.github.io` 仓库），一般留空 `base_path` 即可。

## 工作流文件（.github/workflows/deploy.yml）

推荐的工作流示例（可直接复制使用）：

```yaml
name: Deploy static content to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Private Config
        uses: actions/checkout@v4

      - name: Checkout Dove
        uses: actions/checkout@v4
        with:
          repository: dovenav/dove
          path: dove
          # 可选：固定到某 tag/commit，避免上游变动影响：
          # ref: v0.1.0

      # 安装 Rust 工具链（推荐）
      - name: Setup Rust
        uses: dtolnay/rust-toolchain@stable

      # 缓存 Cargo 依赖与构建产物，加速后续构建
      - name: Cache cargo registry and build
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            dove/target
          key: ${{ runner.os }}-cargo-${{ hashFiles('dove/Cargo.lock') }}
          restore-keys: |
            ${{ runner.os }}-cargo-

      # 拷贝配置文件到 dove 目录
      - name: Copy config file
        run: cp dove.yaml dove/

      # 执行构建
      - name: Build
        run: cd dove && cargo run -- build

      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: 'dove/dist'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> 说明：GitHub 托管的 `ubuntu-latest` 默认不预装 Rust。强烈建议加入 “Setup Rust” 步骤，否则 `cargo` 可能不存在导致构建失败。

### 可选增强

- 固定上游版本：在 “Checkout Dove” 步骤加 `ref: <tag or commit>`，避免上游变更引发构建差异。
- 自定义主题/静态资源：
  - 可在本仓库添加目录（例如 `static/`、`themes/my-theme/`），并调整构建命令，如：
    - `cargo run -- build --static-dir ../static --theme ../themes/my-theme`
  - 或在 `dove.yaml` 中设置 `site.theme_dir` 覆盖默认主题。
- SEO 路径与域名：
  - 项目页（`https://<user>.github.io/<repo>/`）建议设置 `site.base_path: <repo>`。
  - 自定义域名时，建议设置 `site.base_url: https://nav.example.com`，并在仓库 Pages 里配置自定义域。
- 构建加速：已配置 `actions/cache` 缓存 Cargo 依赖与 `dove/target`，以加速二次构建。

## 部署到 Cloudflare Workers（可选）

本仓库已内置基于 Wrangler 的 Workers 部署方案：

- 工作流：`dove-private/.github/workflows/deploy-worker.yml`
- 配置：`dove-private/wrangler.toml`

工作原理与 Pages 相同：在 CI 中 checkout 本仓库与 [dovenav/dove](https://github.com/dovenav/dove)，安装 Rust，复制 `dove.yaml` 后构建到 `dove/dist/`，随后用 Wrangler 将 `dove/dist` 作为静态资源发布到 Workers。

准备工作：

- 在 GitHub 仓库添加 Secrets（Settings → Secrets and variables → Actions）：
  - `CLOUDFLARE_API_TOKEN`：具备“Edit Workers”权限的 API Token。
  - `CLOUDFLARE_ACCOUNT_ID`：Cloudflare 账户 ID（Cloudflare 仪表盘可见）。
- 如需本地直接 `wrangler deploy`，可在 `wrangler.toml` 中填写 `account_id`（默认留空，CI 走 inputs）。

Wrangler 版本：工作流使用 `cloudflare/wrangler-action@v3`。

触发方式：

- 对 `main` 分支的 push 会自动部署；也可在 Actions 中手动运行（`workflow_dispatch`）。

访问地址：

- 默认域名形如 `https://<worker-name>.<你的 workers 子域>.workers.dev/`。
- 若 `dove.yaml` 设置了 `site.base_path: <repo>`，入口页面在 `/<repo>/` 路径下，例如：
  - `https://<worker-name>.<子域>.workers.dev/<repo>/`。
  - 若留空 `base_path`，则首页为根路径 `/`（对应 `dove/dist/index.html`）。

自定义域/路由（可选）：

- 在 Cloudflare 控制台 → Workers & Pages → 选择你的 Worker：
  - 添加 Custom Domain，将你的域名（已托管在 Cloudflare）绑定到该 Worker。
  - 或添加 Route（如 `example.com/*`）映射到该 Worker。
- 使用 `site.base_path` 时，自定义域下的访问路径同样需要带上该子路径。

排错提示：

- 404（根路径为空）：常见于设置了 `site.base_path`，直接访问 `/` 不存在，需访问 `/<base_path>/`。
- 权限错误：确认 `CLOUDFLARE_API_TOKEN` 具备 Workers 相关的编辑权限，且 `CLOUDFLARE_ACCOUNT_ID` 正确。

## 选择 Pages 还是 Workers？

- GitHub Pages：简单、免费，适合托管静态站点（支持自定义域）；无需额外密钥。
- Cloudflare Workers：支持全局加速与路由/自定义域的灵活性；需配置 API Token 与 Account ID。

## 本地调试

- 预览：在本机克隆 [dovenav/dove](https://github.com/dovenav/dove)，将你的 `dove.yaml` 放到仓库根目录，执行：
  - `cargo run -- build`
  - `cargo run -- preview --build-first`（默认 `127.0.0.1:8787`）
- 构建结果位于 `dist/` 或 `dist/<base_path>/`（如果设置了 `base_path`）。

## 配置说明（dove.yaml）

顶层通常包括：

- `site`：站点与页面行为
  - `title`/`description`：标题与描述
  - `color_scheme`：`auto|light|dark`
  - `theme_dir`：主题目录（包含 `templates/` 与 `assets/`）
  - `base_path`：输出到 `dist/<base_path>/` 并作为站点根路径
  - `layout`：`default|ntp`（两种首页布局）
  - `search_engines`/`default_engine`：搜索引擎切换
  - `sitemap`：`default_changefreq`、`default_priority`、`lastmod`
  - `redirect`（外网详情/中间页）：`delay_seconds`、`default_risk`、`utm`
  - 可选统计：`baidu_tongji_id`、`google_analytics_id`
- `groups`：分组与链接
  - `category`：一级分类（侧边栏）
  - `name`：分组标题（内容区）
  - `display`：分组显示风格 `standard|compact|list|text`
  - `links[]`：`name`、`url`、`icon`、`intro`、`details`、（可选）`intranet` 等

提示：

- 内/外网两套页面：外网页面使用 `url`，内网页面优先使用 `intranet`（没有则回退 `url`）。
- 中间页（外网详情）默认开启，可用参数或环境变量禁用：
  - `cargo run -- build --generate-intermediate-page false`
  - `DOVE_GENERATE_INTERMEDIATE_PAGE=false cargo run -- build`
- 配置拆分：支持在主配置中通过 `include` 引用本地片段（支持 glob），并按顺序合并。
- 远程配置：支持从 URL/Gist 加载（需 `--features remote`），详见 dove 主仓库 README。

## 常见问题（FAQ）

- 构建时报 `cargo: command not found`：
  - 在工作流中加入安装 Rust 的步骤，例如 `dtolnay/rust-toolchain@stable`。
- 发布后页面 404 或样式丢失：
  - 检查 `site.base_path` 是否与仓库名一致（项目页场景）。
  - 若使用自定义域，请配置 `site.base_url` 并在 Pages 设置里绑定域名。
- 想把配置放在私有 Gist：
  - dove 支持从远程加载配置（启用 `remote` 特性），可参考主仓库 README 中的 `--input-url`/`DOVE_GIST_ID` 相关说明。

## 给想要自己搭建的人一些建议

- 从本仓库拷贝 `.github/workflows/deploy.yml` 到你的“配置仓库”，仅保留 `dove.yaml` 与工作流即可；不要把 `dove` 源码复制进去。
- 明确 Pages 类型：
  - User/Org Pages（`<username>.github.io`）→ 通常无需设置 `base_path`。
  - Project Pages（`<username>.github.io/<repo>`）→ 建议将 `site.base_path` 设为 `<repo>`。
- 固定依赖版本，避免未来上游变更带来的不可预期差异。
- 先在本地跑通构建与预览，再推上云端；遇到问题用同样的构建命令可快速复现。
- 如需更强的私有化（例如将配置放私有 Gist、或在 CI 中注入 Token），可结合 dove 的远程配置能力与 GitHub Actions 的 `secrets` 使用。

---

当你需要查看更详细的配置字段与命令参数，请前往 [dovenav/dove](https://github.com/dovenav/dove) 主仓库的 README（本仓库仅聚焦 Pages 发布流程）。
