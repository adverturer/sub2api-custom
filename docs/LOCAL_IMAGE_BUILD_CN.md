# Sub2API 自定义版本构建与部署手册

本文记录如何在 Windows + Docker Desktop 环境中，将官方 Sub2API 更新与自己的 UI 改动组合起来，构建 `sub2api:local` 镜像，并替换正在运行的应用容器。

最后更新：2026-07-22

整理依据：Grok CLI 会话 `019f880c-6d76-7622-8087-c10bc17c8848` 中的实际构建记录，以及随后对 Compose 部署、运行镜像和 WSL 端口转发的排查结果。

## 1. 当前目录与仓库约定

本机目前使用两个目录：

| 路径 | 用途 |
| --- | --- |
| `E:\software\Docker\sub2api-src` | 自己 fork 后 clone 的源码仓库；拉取官方更新、开发、提交和构建镜像都在这里进行 |
| `E:\software\Docker\sub2api\deploy` | 当前部署目录；保存 Compose 配置、`.env` 和持久化卷配置 |

源码仓库的远程仓库约定：

```text
origin   -> https://github.com/adverturer/sub2api-custom.git
upstream -> https://github.com/Wei-Shaw/sub2api.git
```

当前自定义 UI：

```text
branch: feature/ui-cache-stats
commit: 94bc082 feat(ui): show cache hit/create stats and hit rate
```

该 UI 在 Usage 统计卡片中增加：

```text
缓存命中: ... · 缓存创建: ...
缓存命中率: 缓存读取/(输入 + 缓存读取 + 缓存创建) ...%
```

命中率分母为：

```text
input_tokens + cache_read_tokens + cache_creation_tokens
```

## 2. 每次构建的推荐顺序

以后需要构建时，按下面的顺序操作：

1. 确认 Git 分支和工作区状态。
2. 确认 Docker Desktop 引擎正在运行。
3. 验证 Docker Hub 拉取能力。
4. 验证构建容器的 DNS。
5. 构建 `sub2api:local`。
6. 检查新镜像的 ID 和创建时间。
7. 用 Compose 只重建 `sub2api` 应用容器。
8. 检查健康状态、日志和容器实际使用的镜像 ID。
9. 浏览器强制刷新，验证 UI。

不要一开始就执行 `docker compose down -v`。`-v` 会删除 Compose 管理的数据卷，可能导致数据库、Redis 或应用数据丢失。

## 3. 构建前检查 Git 状态

进入源码仓库：

```powershell
cd E:\software\Docker\sub2api-src
git status -sb
git branch -vv
```

确认自己正在包含 UI 改动的分支上：

```powershell
git switch feature/ui-cache-stats
```

查看 UI 提交是否存在：

```powershell
git log -5 --oneline --decorate
```

构建使用的是当前工作区内容，包括尚未提交的修改。因此，构建前应确认没有误改文件，也不要把临时日志、PID 文件或临时 Dockerfile 一起提交。

当前目录曾出现这些构建临时文件：

```text
build-local.err
build-local.log
build-local.pid
Dockerfile.local
```

它们不是 UI 功能的一部分。提交时优先使用明确的文件路径，不要直接执行 `git add .`。

## 4. Docker Desktop 与 Clash 代理

### 4.1 什么时候需要代理

构建过程中可能需要访问：

- `auth.docker.io`
- `registry-1.docker.io`
- Docker Hub/CloudFront 镜像层下载地址
- `registry.npmjs.org`
- `goproxy.cn` 或其他 Go Module 代理
- Alpine 软件仓库

如果 Docker Desktop 无法访问这些地址，会在不同构建阶段出现超时。

### 4.2 Clash 推荐状态

如果在 Docker Desktop 中手动填写代理：

- Clash 必须运行。
- Clash 应开启“允许局域网连接”。
- Docker Desktop 使用手动代理时，Windows“系统代理”通常不必开启。
- Clash 平时可使用“规则模式”。
- 排障时可临时切到“全局模式”；全局成功而规则失败，说明规则没有让相关域名走代理。

### 4.3 Docker Desktop 代理地址

Docker Desktop 的代理地址要使用 Windows 主机上、Docker Desktop 能访问的地址。例如：

```text
HTTP:  http://10.148.4.46:7897
HTTPS: http://10.148.4.46:7897
```

其中 `10.148.4.46` 是当时使用的本机地址，切换 Wi-Fi、网线或网络环境后可能改变。不能把它当成永久固定值。

查看本机 IPv4 地址：

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
  Where-Object {
    $_.IPAddress -notlike '127.*' -and
    $_.IPAddress -notlike '169.254.*'
  } |
  Select-Object InterfaceAlias,IPAddress
```

测试 Clash 端口是否监听：

```powershell
Test-NetConnection -ComputerName 10.148.4.46 -Port 7897
```

`TcpTestSucceeded` 应为 `True`。

不要默认填写 `127.0.0.1:7897`。Docker Desktop 的 Linux 环境里，`127.0.0.1` 不一定指向 Windows 主机。

`host.docker.internal` 在部分 Docker Desktop 网络阶段可能无法解析。如果它报 `no such host`，改用当前 Windows 主机 IPv4 地址。

修改代理后点击 Docker Desktop 的 **Apply**，等待 Engine 完成重启。

### 4.4 如何确认代理设置已写入

Docker Desktop 设置保存在：

```text
C:\Users\幻梦\AppData\Roaming\Docker\settings-store.json
```

只查看代理相关字段：

```powershell
$settings = Get-Content "$env:APPDATA\Docker\settings-store.json" -Raw |
  ConvertFrom-Json

$settings | Select-Object `
  ProxyHTTPMode,OverrideProxyHTTP,OverrideProxyHTTPS,OverrideProxyExclude
```

`OverrideProxyHTTP`/`OverrideProxyHTTPS` 可能保留上一次填写的值，因此还要同时查看 `ProxyHTTPMode`。如果模式是 `disabled`，仅看到 7897 地址并不代表手动代理当前正在启用。

`docker info` 可能始终显示：

```text
HTTP Proxy:  http.docker.internal:3128
HTTPS Proxy: http.docker.internal:3128
```

这是 Docker Desktop 内部的 hub proxy 入口，不等于手动填写的 7897 没有生效。最终应以实际拉取结果为准。

### 4.5 最关键的代理验证

执行：

```powershell
docker pull docker/dockerfile:1.7
```

如果成功或显示镜像已经是最新版本，说明最初失败的 Dockerfile frontend 已经可以访问。

也可以测试：

```powershell
docker run --rm hello-world
```

## 5. Dockerfile 第一行的说明

官方 Dockerfile 第一行是：

```dockerfile
# syntax=docker/dockerfile:1.7
```

这不是普通注释。BuildKit 会先拉取 `docker/dockerfile:1.7` 作为 Dockerfile frontend。因此即使 Node、Go、Alpine 等基础镜像已经缓存，本次构建仍可能在第一行失败。

典型错误：

```text
resolve image config for docker-image://docker.io/docker/dockerfile:1.7
failed to fetch anonymous token
auth.docker.io ... timeout
```

长期正确做法是修好 Docker Desktop 代理并保留官方第一行。

当时为了绕过 frontend 拉取，曾临时改成：

```dockerfile
## syntax=docker/dockerfile:1.7
```

这只应视为网络排障的临时方案，不应作为 UI 功能提交，也不应长期偏离官方 Dockerfile。代理恢复后，将它恢复成官方的单个 `#`，再运行标准构建。

## 6. Docker 构建容器的 DNS

如果已经能拉基础镜像，但构建失败在：

```text
go mod download
lookup goproxy.cn on 192.168.65.7:53: i/o timeout
```

说明不是 Go 代码错误，而是 Docker Desktop 虚拟网络里的 DNS 超时。

先测试：

```powershell
docker run --rm alpine:3.21 nslookup goproxy.cn
```

成功时会返回 IPv4/IPv6 地址。失败时，可以在 Docker Desktop 的 **Settings -> Docker Engine** 中，将 `dns` 合并到现有 JSON：

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "dns": ["223.5.5.5", "119.29.29.29", "8.8.8.8"]
}
```

不要覆盖或删除原有字段。Apply & Restart 后再次执行 `nslookup`，确认成功后再构建。

如果 `go mod download` 报 `unexpected EOF`，通常是依赖下载中断。Dockerfile 已使用 Go Module cache mount，网络恢复后直接重试即可，不要立即清空全部构建缓存。

## 7. 构建本地镜像

进入源码仓库：

```powershell
cd E:\software\Docker\sub2api-src
```

先看当前改动：

```powershell
git status -sb
git diff --stat
```

标准构建命令：

```powershell
docker build --progress=plain -t sub2api:local .
```

普通界面输出也可以：

```powershell
docker build -t sub2api:local .
```

不要日常使用 `--no-cache`。pnpm 和 Go Module 缓存可以显著减少重试成本。只有确定缓存层损坏时再考虑清缓存。

如果 npm 官方源不稳定，Dockerfile 支持通过构建参数指定 registry：

```powershell
docker build `
  --build-arg NPM_CONFIG_REGISTRY=https://registry.npmmirror.com `
  -t sub2api:local .
```

Dockerfile 默认 Go 配置为：

```text
GOPROXY=https://goproxy.cn,direct
GOSUMDB=sum.golang.google.cn
```

必要时可以显式传入：

```powershell
docker build `
  --build-arg GOPROXY=https://goproxy.cn,direct `
  --build-arg GOSUMDB=sum.golang.google.cn `
  -t sub2api:local .
```

成功时最后应看到类似：

```text
FINISHED
naming to docker.io/library/sub2api:local
```

## 8. 验证镜像

查看镜像：

```powershell
docker image ls sub2api
```

查看精确 ID 和创建时间：

```powershell
docker image inspect sub2api:local `
  --format 'ID={{.Id}} Created={{.Created}} Tags={{json .RepoTags}}'
```

建议为成功版本保留一个不会被下一次构建覆盖的标签：

```powershell
docker tag sub2api:local sub2api:2026-07-22-ui1
```

以后可以按日期和自己的补丁号命名，例如：

```text
sub2api:2026-08-05-ui2
sub2api:v1.2.3-custom.1
```

## 9. 使用新镜像替换应用容器

当前部署目录：

```powershell
cd E:\software\Docker\sub2api\deploy
```

Compose 中应用服务名是 `sub2api`，应使用：

```yaml
services:
  sub2api:
    image: sub2api:local
```

先检查 Compose 最终配置：

```powershell
docker compose config
```

重点确认：

- `services.sub2api.image` 是 `sub2api:local`。
- 端口是宿主机 `8080` 到容器 `8080`。
- PostgreSQL、Redis 和 Sub2API 的数据卷仍然存在。

只替换应用容器，不重建数据库和 Redis：

```powershell
docker compose up -d `
  --no-deps `
  --force-recreate `
  --pull never `
  sub2api
```

参数说明：

| 参数 | 作用 |
| --- | --- |
| `--no-deps` | 不重建 PostgreSQL 和 Redis |
| `--force-recreate` | 即使配置未变化，也重新创建应用容器 |
| `--pull never` | 只使用本地 `sub2api:local`，不去远程拉取 |

这条命令不会删除数据卷。

## 10. 验证运行容器确实用了新镜像

检查状态：

```powershell
docker compose ps
```

检查日志：

```powershell
docker compose logs --tail 100 sub2api
```

检查容器状态和镜像名：

```powershell
docker inspect sub2api `
  --format 'ImageName={{.Config.Image}} ImageID={{.Image}} Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'
```

比较容器镜像 ID 和本地标签 ID：

```powershell
$localImage = docker image inspect sub2api:local --format '{{.Id}}'
$containerImage = docker inspect sub2api --format '{{.Image}}'

"local:     $localImage"
"container: $containerImage"
"same:      $($localImage -eq $containerImage)"
```

`same` 应为 `True`。

检查健康接口：

```powershell
curl.exe --noproxy "*" --max-time 10 http://127.0.0.1:8080/health
```

正常结果：

```json
{"status":"ok"}
```

最后打开：

```text
http://localhost:8080
```

使用 `Ctrl + F5` 强制刷新，避免浏览器继续使用旧的前端静态资源。

## 11. localhost 访问卡住与 wslrelay 端口冲突

曾出现以下现象：

- `docker compose ps` 显示 `sub2api` 为 `healthy`。
- 容器内部 `/health` 立即返回正常。
- Windows 能连接 `127.0.0.1:8080`，但 HTTP 请求一直不返回。
- 通过本机局域网 IP 访问 `8080` 却正常。

当时的监听状态是：

```text
127.0.0.1:8080 -> wslrelay.exe
0.0.0.0:8080   -> com.docker.backend.exe
```

`wslrelay.exe` 是 WSL 自带的 localhost 端口转发组件。即使没有手动打开 Linux 发行版，Docker Desktop 使用 WSL 2 后端时也会自动启动 `docker-desktop` 和相关转发进程。

检查端口：

```powershell
$listeners = Get-NetTCPConnection `
  -LocalPort 8080 `
  -State Listen `
  -ErrorAction SilentlyContinue

$listeners | Format-Table LocalAddress,LocalPort,OwningProcess

$listeners.OwningProcess |
  Sort-Object -Unique |
  ForEach-Object { Get-Process -Id $_ | Select-Object Id,ProcessName,Path }
```

区分应用故障和 Windows 转发故障：

```powershell
# 容器内部
docker exec sub2api wget -q -T 5 -O - http://127.0.0.1:8080/health

# Windows 本机，明确绕过代理
curl.exe --noproxy "*" --max-time 10 http://127.0.0.1:8080/health
```

如果容器内部正常、Windows 本机超时，可以完整重置一次 WSL/Docker 网络：

1. 从系统托盘完全退出 Docker Desktop。
2. 执行：

```powershell
wsl --shutdown
```

3. 确认 `127.0.0.1:8080` 的异常 `wslrelay` 监听消失。
4. 重新启动 Docker Desktop。
5. 等待 Engine Running，再重新检查 Compose 和 `/health`。

`wsl --shutdown` 不会删除镜像、容器和数据卷。

如果冲突反复出现，可以把宿主机端口改成其他未占用端口，例如直接将 Compose 映射改为：

```yaml
ports:
  - "8081:8080"
```

然后访问 `http://localhost:8081`。

## 12. 常见错误快速对照

| 报错或现象 | 原因 | 处理 |
| --- | --- | --- |
| `resolve image config for docker-image://docker.io/docker/dockerfile:1.7` | BuildKit 无法访问 Docker Hub frontend | 检查 Clash、Docker Desktop 手动代理；用 `docker pull docker/dockerfile:1.7` 验证 |
| `failed to fetch anonymous token` / `auth.docker.io timeout` | Docker Hub 鉴权请求没有走通 | 检查代理地址、Clash 允许局域网、规则/全局模式 |
| `docker info` 仍显示 `http.docker.internal:3128` | Docker Desktop 内置代理入口 | 属于正常架构；查看 `settings-store.json` 并以实际 `docker pull` 为准 |
| `host.docker.internal: no such host` | Docker 当前网络阶段无法解析该主机名 | Docker Desktop 代理改用当前 Windows IPv4 地址 |
| `lookup goproxy.cn on 192.168.65.7:53: i/o timeout` | Docker 虚拟网络 DNS 超时 | Docker Engine 增加 DNS；用 Alpine `nslookup` 验证 |
| Go Module `unexpected EOF` | 下载途中断开 | 网络恢复后直接重试，保留 cache mount |
| pnpm `ETIMEDOUT` | npm 下载链路不稳定 | 检查代理，或传入 `NPM_CONFIG_REGISTRY` |
| 容器 healthy，但 localhost 一直加载 | Windows/WSL 端口转发异常 | 检查 `wslrelay`，退出 Docker Desktop 后 `wsl --shutdown` |
| 页面还是旧 UI | 运行的容器仍是旧镜像，或浏览器缓存 | 比较镜像 ID；强制 recreate；浏览器 `Ctrl + F5` |
| Compose 去拉官方镜像 | 使用了错误 Compose 文件，或 `image` 仍是官方地址 | 执行 `docker compose config`，确认 `sub2api:local`，部署时加 `--pull never` |

## 13. 将 UI 改动保存到自己的 GitHub fork

### 13.1 检查提交内容

```powershell
cd E:\software\Docker\sub2api-src
git switch feature/ui-cache-stats
git status -sb
git log --oneline main..HEAD
git diff --stat main...HEAD
```

UI 提交应为：

```text
94bc082 feat(ui): show cache hit/create stats and hit rate
```

仅 UI 提交涉及：

```text
frontend/src/components/admin/usage/UsageStatsCards.vue
frontend/src/components/admin/usage/__tests__/UsageStatsCards.spec.ts
```

### 13.2 推送功能分支

首次推送：

```powershell
git push -u origin feature/ui-cache-stats
```

以后该分支有新提交时：

```powershell
git push
```

截至 2026-07-22，远程 `origin/feature/ui-cache-stats` 已经存在，并指向 UI 提交 `94bc082`。也就是说 UI 代码已经上传到自己的 fork；剩余工作是把它合并进自己 fork 的 `main`。

### 13.3 推荐通过 GitHub Pull Request 合并

打开自己的仓库：

```text
https://github.com/adverturer/sub2api-custom
```

创建 Pull Request：

- base：`main`
- compare：`feature/ui-cache-stats`
- 标题：`feat(ui): show cache hit/create stats and hit rate`

确认 PR 只包含预期文件后合并。这样 GitHub 上会保留功能讨论、文件差异和合并记录。

合并后同步本地 `main`：

```powershell
git switch main
git pull --ff-only origin main
```

在当前工作区仍有未提交的 Dockerfile 修改时，不要急着切换分支或执行 `git add .`。先确认这些临时构建修改是否需要保留，并确保它们不会混入 UI 提交。

## 14. 后续同步官方更新

每次准备跟随官方版本时：

```powershell
cd E:\software\Docker\sub2api-src
git fetch upstream
git switch main
git merge --no-ff --no-commit upstream/main
git commit -m "chore(sync): merge upstream main"
git push origin main
```

自己的 `main` 合入 UI 后会与官方主线分叉，此时应使用普通 merge，而不是 `--ff-only`。`--no-commit` 允许在提交前先解决冲突和验证；提交时使用 `chore(sync): merge upstream main`，保证合并记录符合 Conventional Commits。

如果 Git 提示 `Already up to date.`，说明暂时没有新的官方提交，此时不需要执行后面的 `git commit`。

然后让自己的 UI 分支跟上新主线：

```powershell
git switch feature/ui-cache-stats
git rebase main
```

若发生冲突，重点检查 UI 修改的两个文件。解决后：

```powershell
git add frontend/src/components/admin/usage/UsageStatsCards.vue
git add frontend/src/components/admin/usage/__tests__/UsageStatsCards.spec.ts
git rebase --continue
```

如果该功能分支已经公开推送，rebase 后需要谨慎更新远程历史。更稳妥的长期方式是：UI 合并进自己的 `main` 后，在每次官方更新时直接把 `upstream/main` 合并到自己的 `main`，通过普通 merge 解决冲突，避免反复改写公开分支历史。

新增自定义功能时继续使用 Conventional Commits，例如：

```text
feat(ui): add cache usage summary
fix(ui): handle empty cache token totals
docs(build): update local image build guide
```

## 15. 一页命令清单

网络预检：

```powershell
docker pull docker/dockerfile:1.7
docker run --rm alpine:3.21 nslookup goproxy.cn
```

构建：

```powershell
cd E:\software\Docker\sub2api-src
git status -sb
docker build --progress=plain -t sub2api:local .
docker image inspect sub2api:local --format 'ID={{.Id}} Created={{.Created}}'
```

部署：

```powershell
cd E:\software\Docker\sub2api\deploy
docker compose config
docker compose up -d --no-deps --force-recreate --pull never sub2api
docker compose ps
docker compose logs --tail 100 sub2api
```

验证：

```powershell
$localImage = docker image inspect sub2api:local --format '{{.Id}}'
$containerImage = docker inspect sub2api --format '{{.Image}}'
"same: $($localImage -eq $containerImage)"

curl.exe --noproxy "*" --max-time 10 http://127.0.0.1:8080/health
```

Git：

```powershell
cd E:\software\Docker\sub2api-src
git switch feature/ui-cache-stats
git status -sb
git push -u origin feature/ui-cache-stats
```

## 16. 安全提醒

- `deploy\.env` 中包含数据库密码、Redis 密码、管理员密码、JWT Secret 和 TOTP 密钥，不要提交到公开仓库。
- 分享 `docker compose config` 输出前，应先把密码和密钥替换为 `***`。
- 不要使用 `docker compose down -v` 作为普通重启命令。
- 不要把临时 Dockerfile 绕过、构建日志和 PID 文件混进 UI 提交。
- 每次部署后都比较本地镜像 ID 与容器镜像 ID，不要只看标签名。
