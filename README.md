# docker-image-packer

一个 GitHub Actions 工作流：在云端把 Docker Hub / GHCR 的镜像拉取（或源码构建）后，
`docker save` 成 tar、压缩为 zip，并**发布到 GitHub Releases**，供离线环境下载导入。

```bash
# 离线端使用（先解压 zip 得到 .tar）
docker load -i xxxx.tar
```

## 功能

- **全部打包**：解析 `images.txt`，逐镜像并行拉取/构建、独立打包。
- **单镜像打包**：手动触发时指定一个镜像名，只打包它。
- **发布 Releases**：所有 zip 发布为一个 Release，标签为 `pack-<运行编号>`。
- **大镜像自动分片**：超过 GitHub 单 asset 2 GiB 上限的镜像（如 ollama）自动拆分为 1 GiB 分卷上传。
- **旧版自动清理**：新 Release 发布成功后，自动删除历史 `pack-*` 与遗留 `latest` 版。
- **gh-proxy 加速链接**：Release Notes 和运行摘要中同时给出原始下载链接与 `gh-proxy.com` 加速链接，便于国内网络下载。
- **Docker Hub 限流规避**：支持可选登录态拉取（高限额），无凭据时自动退回匿名并对拉取做指数退避重试。
- **追踪上游最新**：`images.txt` 使用 `latest` / `stable` 追踪 tag，每次打包拉取上游当前最新版本。

## 使用方法

### 1. 编辑镜像清单 `images.txt`

每行一个镜像，`#` 开头为注释，支持两种写法：

```
# 直接拉取（Docker Hub 或 GHCR），用最新/稳定追踪 tag
ollama/ollama:latest
ghcr.io/home-assistant/home-assistant:stable

# 源码构建
build:https://github.com/xiasi0/wyoming-sherpa-onnx.git|wyoming-sherpa-onnx:local
```

追踪 tag 约定：

- `latest`：Docker Hub 的浮动 tag，指向最近一次推送，适合 `rhasspy/*`、`ollama/ollama` 等。
- `stable`：Home Assistant 官方镜像（`ghcr.io/home-assistant/home-assistant`）与
  `ghcr.io/hasscc/hacn` 的"最新稳定版"追踪 tag，比 `latest` 语义更明确。

> **注意**：追踪 tag 每次打包都会拉取上游最新内容，同一镜像两次打包的结果可能不同。
> 这符合"始终用最新版"的诉求——只要每一次都重新触发打包即可拿到最新。

### 2. （可选）配置 Docker Hub 登录，规避限流

GitHub 的共享 runner 出口 IP 使用 Docker Hub 匿名拉取时，容易命中 ~100 pulls/6h/IP 的限流，
导致 `docker pull` 失败。推荐在仓库 **Settings → Secrets and variables → Actions** 配置两个 secret：

| Secret 名称 | 值 |
| :--- | :--- |
| `DOCKERHUB_USERNAME` | 你的 Docker Hub 用户名 |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token（建议用 token 而非密码） |

- 配置后，workflow 会自动用登录态拉取（享受更高限额）。
- 未配置时，自动退回匿名拉取，并对每次 pull 做指数退避重试（5s → 10s → 20s → 40s，最多 4 次），
  以尽量扛过瞬时限流与网络抖动。

### 3. 手动触发打包

在仓库 **Actions** 页选择「Pack Docker Images」→ **Run workflow**：

- 输入 `all`（默认）：打包 `images.txt` 里的全部镜像。
- 输入具体镜像名（如 `ollama/ollama:latest`）：只打包该单个镜像。

### 4. 下载

打包完成后，到 **Releases** 页的 `pack-<编号>` 版本下载对应 zip（国内网络可用 Release Notes 里的 gh-proxy 加速链接）。

- 小于 2 GiB 的镜像：直接解压 zip 得 `.tar`。
- 超过 2 GiB 的镜像（如 ollama）：已拆分为 `.part000` / `.part001` … 分卷，先合并再解压：

```bash
# Linux / macOS / Git Bash
cat ollama_ollama_latest.zip.part* > ollama_ollama_latest.zip

# Windows CMD
copy /b ollama_ollama_latest.zip.part000 + ollama_ollama_latest.zip.part001 ollama_ollama_latest.zip
```

最后离线导入：

```bash
docker load -i xxxx.tar
```

## 说明

- 打包产物走 GitHub Artifacts（保留 7 天）+ GitHub Releases 双通道，Releases 长期有效。
- Release 标签为 `pack-<运行编号>`，每次发布后自动清理旧版，仅保留最新一次。
- 采用「先发布新版、成功后再删旧版」的顺序，避免上传失败导致旧版丢失。
- Artifacts 保留 7 天仅为临时兜底；正式分发请以 Releases 为准。