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
- **发布 Releases**：所有 zip 发布为一个 Release，标签固定为 `latest`。
- **旧版自动清理**：每次发布前删除上一个 `latest` Release，只保留最新一次。
- **gh-proxy 加速链接**：Release Notes 和运行摘要中同时给出原始下载链接与 `gh-proxy.com` 加速链接，便于国内网络下载。
- **Docker Hub 限流规避**：支持可选登录态拉取（高限额），无凭据时自动退回匿名并对拉取做指数退避重试。
- **镜像版本固定**：`images.txt` 默认固定到具体版本号，保证两次打包结果一致、可复现。

## 使用方法

### 1. 编辑镜像清单 `images.txt`

每行一个镜像，`#` 开头为注释，支持两种写法：

```
# 直接拉取（Docker Hub 或 GHCR），建议固定版本号保证可复现
ollama/ollama:0.34.0
ghcr.io/home-assistant/home-assistant:2026.9.2

# 源码构建
build:https://github.com/xiasi0/wyoming-sherpa-onnx.git|wyoming-sherpa-onnx:local
```

> **重要**：请把镜像 tag 固定为具体版本号，而非 `latest`。`latest` 是浮动 tag，
> 两次打包导出的内容可能不同，会导致离线分发不可复现。

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

打包完成后，到 **Releases** 页的 `latest` 版本下载对应 zip（国内网络可用 Release Notes 里的 gh-proxy 加速链接），
解压得到 `.tar` 后离线导入：

```bash
docker load -i xxxx.tar
```

## 说明

- 打包产物走 GitHub Artifacts（保留 7 天）+ GitHub Releases 双通道，Releases 长期有效。
- 采用**覆盖式发布**（固定 `latest` tag，发布前删除旧版），因此每个仓库同一时间只有一个 Release。
- Artifacts 保留 7 天仅为临时兜底；正式分发请以 Releases 为准。