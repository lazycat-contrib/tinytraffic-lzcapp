# tinytraffic-lzcapp

[egoist/tinytraffic](https://github.com/egoist/tinytraffic) 的懒猫微服打包，
仅发布到**喵喵（私有）商店**，镜像模式（不复制镜像、只改运行时地址）。

**tinytraffic** 是隐私友好的网站访问统计：单个 Go 二进制 + 一个 DuckDB 文件，
约 1 KB 的埋点脚本，不用 Cookie、不存 IP，事件格式兼容 Plausible。

## 打包内容

| 项目 | 值 |
|------|----|
| 包名 | `community.lazycat.app.tinytraffic` |
| 上游镜像 | `ghcr.io/egoist/tinytraffic`（镜像模式） |
| 运行时加速 | `ghcr.1ms.run`（Action 默认 GHCR 加速地址） |
| 版本来源 | 上游三段 SemVer 镜像标签（`0.2.1`，另有 `0.1`/`latest`/`main`/`sha-*` 被过滤掉） |
| 容器端口 | 8080 |
| 运行身份 | `run_as: "65532:65532"`（distroless `nonroot`） |
| 持久化 | `/lzcapp/var/data`（`tinytraffic.duckdb`） |
| 部署参数 | 看板口令（预置随机值，必填） |

## 上游镜像标签

上游 `docker.yml` 的产出是这样的：`main` 分支推送出 `main` 和 `sha-xxxxxxx`，
Git tag `v0.2.1` 出 `0.2.1`、`0.2` 和 `latest`。所以：

- `channel: stable` + `tag_regex: '^v?[0-9]+\.[0-9]+\.[0-9]+$'` 只收三段 SemVer，
  丢掉会漂移的 `latest` / `main` / `sha-*` 和两段的 `0.2`；
- `delivery.mode: mirror` + `require_digest_match: true`，加速地址必须给出
  与上游一致的 amd64 摘要才继续（已核对 `0.2.1` 的 amd64 子摘要
  `sha256:fd1697d7…` 与 `ghcr.io` 相同）。

## 为什么是这个样子

**`TT_TRUST_PROXY=true` 是必需的，不是可选优化。**
懒猫的 `lzc-ingress` 是反向代理，不开这个开关时上游只会看到代理那一个 IP：
独立访客数会塌成 1，Locations 全部是 `Unknown`。开启后按
`CF-Connecting-IP` / `X-Real-IP` / `X-Forwarded-For` 取真实访客 IP。

**只放开采集必需的三条路径：**

```yaml
public_path:
  - /js/script.js   # 访客网站要能把脚本拉下来
  - /api/event      # 访客浏览器直接 POST 上报
  - /api/v1         # 带 API Key 的读数/写数接口
```

看板、站点管理、改密接口都在平台入口登录后面。`/api/v1` 由应用自己的
`Bearer tt_…` API Key 鉴权，且只能读统计、管站点，碰不到本机文件。

**没有容器健康检查。** 运行时镜像是 `gcr.io/distroless/cc-debian12:nonroot`，
里面没有 shell、也没有 curl/wget，写 `CMD-SHELL` 健康检查只会让容器反复重启；
上游 Dockerfile 本身也没有 `HEALTHCHECK`。平台会按 upstream 声明的 8080
端口做自动监测。（应用其实有 `GET /healthz`，将来上游若换成带 shell 的
基础镜像可以再补上。）

**`run_as: "65532:65532"`。** 镜像以 `nonroot`（UID/GID 65532）运行，
`/lzcapp/var/data` 需要按同一身份暴露 owner，否则 DuckDB 建不了库。
`run_as` 要求 lzcos ≥ 1.6.0，所以 `min_os_version` 也提到 `1.6.0`；
它不能与 `user` / `setup_script` 同时使用，这里都没有用。

**口令与免密登录。** 上游是「单口令」登录（登录页只有密码框），
首启时若没有任何用户就会按 `TT_PASSWORD` 直接建好管理员，
别人抢不到 setup 页。口令来自部署参数（默认 `$random(len=20)`），
再配三段式 inject：`request` 抓候选口令 → `response` 仅在 200 时
`ctx.persist` → `browser` 在 `/login` 自动填充并提交。
用户在应用里改过口令后，填充跟着 persist 的最新值走。

**没有做任何空格路径的绕行。** 上游数据目录是 `TT_DATA_DIR=/data`
（Dockerfile 里已经写死），不像 Hound 那种相对工作目录的 `Hound Data`，
所以直接绑 `/lzcapp/var/data:/data` 就行。

## 版本与发布

- `version_source` 指向 `tinytraffic` 镜像，每天定时检查上游新标签。
- 新版本会更新 `package.yml` 的 `version` 与 `lzc-manifest.yml` 的镜像地址，
  构建 LPK、创建带版本号的 Release 资产，再提交到喵喵商店。
- 官方商店关闭（`stores.official.enabled: false`），不会触发官方审核。

## 许可与版权

上游仓库**目前没有 LICENSE 文件**（GitHub API 识别为 `No license`），
`package.yml` 因此写 `license: NOASSERTION`。本仓库只做懒猫打包，
按镜像模式引用作者公开发布的容器镜像，不复制、不修改、不重分发源码；
商用或再分发请先与作者确认授权。

镜像里另外带了 [DB-IP](https://db-ip.com) 免费城市库（CC BY 4.0）做
IP 归属地，应用内的 Locations 卡片会按它的许可要求显示署名。

本包尚未在懒猫实机安装验证。
