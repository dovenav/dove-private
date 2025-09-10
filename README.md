# dove-private

基于 [dove](https://github.com/dovenav/dove) 的个人（或组织）导航站点私有配置仓库。通过 GitHub Actions 自动构建并发布到 GitHub Pages。

本仓库只存放你的站点配置（`dove.yaml` 等），实际构建时会从 GitHub 拉取 [dovenav/dove](https://github.com/dovenav/dove) 源码，在 CI 里执行 `cargo run -- build` 产出静态文件并发布。

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

## 本地调试

- 预览：在本机克隆 [dovenav/dove](https://github.com/dovenav/dove)，将你的 `dove.yaml` 放到仓库根目录，执行：
  - `cargo run -- build`
  - `cargo run -- preview --build-first`（默认 `127.0.0.1:8787`）
- 构建结果位于 `dist/` 或 `dist/<base_path>/`（如果设置了 `base_path`）。

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
