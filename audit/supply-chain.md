# 供应链

审计对象：https://github.com/vxcontrol/pentagi ，基准 `cffed9485aa1cca70762cd08a57ac62a80406258`（2026-10-23 07:20 香港时间）。只读。OSV 查询时间为 2026-11-26 10:30 香港时间前后。未拉取镜像层，未执行 `go mod download`、`pnpm install` 或仓库里的脚本。

结论：抽查范围内没有 OSV 恶意包（MAL）记录，也没有看到仿冒主流包名的 require；不能据此说依赖无恶意。直接依赖里有已公开漏洞公告，版本落在受影响区间，可达性未验证。运行时镜像多数只有浮动 tag、没有 digest。

## 依赖是否恶意

结论：未发现已证实的恶意依赖。下列各项要么是普通漏洞公告，要么是未核实。

- 本仓没有 `.gitmodules`。`backend/go.mod` 没有 `replace` 指向外部 fork。厂商模块以正常 require 出现：`github.com/vxcontrol/cloud` v1.0.1、`github.com/vxcontrol/langchaingo` v0.1.14-update.8、`github.com/vxcontrol/graphiti-go-client` v0.9.0。langchaingo 是厂商 fork，不是差一个字符的仿冒名。这三份模块源码不在本仓，OSV 对这三个精确版本返回 0 条，未审其仓库历史，未核实是否恶意。
- `backend/go.mod` 直接 require 约 77 条（第一段加间接段之前的计数含直接依赖；精确分界以文件为准）。`backend/go.sum` 以模块路径开头的行约 883 条，含各模块的 `.mod` 行。只核对了 go.sum 存在且非空，没有逐条打开校验和，也没有把每一条送进恶意包库。
- `frontend/package.json` 的 scripts 没有 `preinstall` / `install` / `postinstall`。包管理器字段为 `pnpm@11.8.0`。`frontend/pnpm-lock.yaml` 存在。没有审计锁文件里的全部传递依赖，也没有在注册表上核对每个包的发布者。
- 2026-11-26 10:30 香港时间，对 OSV `https://api.osv.dev/v1/querybatch` 做了只读查询。精确版本与结果：
  - 0 条：`github.com/vxcontrol/cloud@1.0.1`、`github.com/vxcontrol/langchaingo@0.1.14-update.8`、`github.com/vxcontrol/graphiti-go-client@0.9.0`、`github.com/gin-gonic/gin@1.10.0`、`github.com/golang-jwt/jwt/v5@5.2.2`、`github.com/jinzhu/gorm@1.9.16`、`github.com/99designs/gqlgen@0.17.57`、`github.com/jackc/pgx/v5@5.9.2`、`graphql@16.14.2`、`@apollo/client@4.2.10`、`react@19.2.7`。
  - 有公告、ID 均不是 MAL：`axios@1.18.0`（12 条 GHSA，锁文件与 `frontend/src/lib/axios.ts` 对得上）、`golang.org/x/crypto@0.55.0`（3 条，子包 ssh/openpgp，本仓 Go 源码无直接 import）、`golang.org/x/net@0.58.0`（7 条，`backend/pkg/tools/searchers/duckduckgo.go` 直接 import `golang.org/x/net/html`）、`github.com/ollama/ollama@0.32.5`（9 条，本仓只看到 `api` 与 `types/model` 引用）、`vite@8.0.16`（5 条，锁文件版本；摘要指向 Vite 开发服务器与 Windows 上的 launch-editor，生产镜像阶段跑的是 `pnpm run build`，未确认这些公告进入运行时）。
- 曾误用 vite 7.3.6 查询，该版本不是锁文件版本，结果作废，以 8.0.16 为准。
- 未查询：go.sum 与 pnpm-lock 的其余传递依赖、GHSA 网页上的每一条评论、Socket/deps.dev 的维护者信誉。查不到的一律未核实，不写成无恶意。
- `github.com/jinzhu/gorm` v1.9.16 仍被服务层使用。它是已停止开发的旧 ORM，不是这次 OSV 结果里的恶意包。

已知漏洞的产品内位置与建议见 `audit/findings.json` 的 F-09。这里不把公告复述成已在本进程里被利用。

## 容器镜像

结论：本单没有拉取任何镜像层，不能说层内无恶意或无漏洞。tag 都没有 digest。

| 位置 | 镜像 | tag | digest |
|---|---|---|---|
| Dockerfile 第 6 行 | `node` | `24.17.0-slim` | 无 |
| Dockerfile 第 55 行 | `golang` | `1.26-bookworm` | 无。go.mod 写的是 `go 1.26.5`，tag 会随 1.26 次版本浮动 |
| Dockerfile 第 136 行 | `alpine` | `3.23.5` | 无 |
| docker-compose.yml 第 26 行 | `vxcontrol/pentagi` | `latest` | 无 |
| docker-compose.yml 第 242 行 | `vxcontrol/pgvector` | `latest` | 无 |
| docker-compose.yml 第 270 行 | `quay.io/prometheuscommunity/postgres-exporter` | `v0.16.0` | 无 |
| docker-compose.yml 第 291 行 | `vxcontrol/scraper` | `latest` | 无 |
| config.go 第 66–72 行 | `debian`、`vxcontrol/kali-linux`、`vxcontrol/kali-linux` | `latest`、无 tag、`test` | 无。agent 还可能 pull 允许列表之外的镜像，见 F-04 |
| docker-compose-graphiti.yml | `neo4j:5.26.2`、`vxcontrol/graphiti:latest` | 见文件 | 无 |
| docker-compose-langfuse.yml | `langfuse/langfuse-worker:3`、`langfuse/langfuse:3`、`clickhouse/clickhouse-server:24`、`cgr.dev/chainguard/minio:latest`、`redis:7`、`postgres:16` | 主版本或 latest | 无 |
| docker-compose-observability.yml | grafana 11.4.0、node-exporter v1.8.2、cadvisor v0.51.0、otel collector 0.116.1、victoria-metrics v1.108.1、clickhouse 24、loki 3.3.2、jaeger 1.56.0 | 见文件 | 无 |
| docker-compose.e2e.yml | `node:24-alpine` | 浮动次版本 | 无。e2e 专用，不是默认栈 |

Dockerfile 第 90 行在构建时 `go install github.com/google/go-licenses@latest`，工具本身不钉版本。`apt-get install` 也未钉包版本。这两步只在构建镜像时发生，本单未构建。

## 安装脚本与钩子

结论：默认产品路径没有发现 `curl | sh` 被仓库脚本执行。文档里有无校验和的下载，可选指南里有管道安装。

- `frontend/package.json` 无安装钩子。
- README.md 第 614 行用 wget 下载 `https://pentagi.com/downloads/linux/amd64/installer-latest.zip`，附近片段没有校验和。第 864 行 `curl -O` 拉取 `https://raw.githubusercontent.com/vxcontrol/pentagi/master/docker-compose.yml`。2026-11-26 10:28 香港时间 `git ls-remote` 只返回 `refs/heads/main` 指向本单 SHA，没有返回 `refs/heads/master`。该 raw URL 是否仍被 GitHub 别名到 main，本单未请求文件正文，标未核实。
- `examples/guides/vllm-qwen35-27b-fp8.md` 第 74 行是 `curl -LsSf https://astral.sh/uv/install.sh | sh`。这是可选本地模型指南，不是 `docker compose` 默认路径。
- `examples/reports/` 里有模型输出的反弹命令文本。它不是安装脚本，构建文件没有调用它。
- `.github/workflows/` 里的 Actions 使用 `actions/checkout@v6`、`actions/setup-go@v6`、`docker/build-push-action@v6` 这类主版本 tag，没有钉 commit SHA。未核实这些 tag 当前指向的提交。

## 未覆盖

- go.sum 与 pnpm-lock 的传递依赖全量恶意包比对。
- 所有镜像的层、基础镜像里的包、vxcontrol 三个自建镜像的 Dockerfile（不在本仓）。
- `github.com/vxcontrol/cloud` 收到的更新请求在 SDK 内部是否还附带别的字段。本仓能看到的请求体是 `ProductUpdateRequest`（`backend/pkg/server/update/update.go` 第 243–260 行）。
- 安装包 zip 与远程 compose 文件的实际字节。未下载。
