# 供应链补查 v4

基准 SHA `cffed9485aa1cca70762cd08a57ac62a80406258`（上游 https://github.com/vxcontrol/pentagi `main`，2026-10-23 07:20 香港时间）。只读解析清单并查询公开索引。未安装依赖，未拉镜像层，未运行仓库脚本或 CI。已有结论见 `audit/supply-chain.md`，这里只补缺口。命中漏洞公告只说明锁版本落在公告范围内，调用路径是否可利用一律未核实。

| 项 | 结论 |
|---|---|
| 1 依赖清单与 lockfile | 已确认 |
| 2 版本钉、校验和、git／URL 直装 | 已确认 |
| 3 仿冒包名、安装钩子、非官方源 | 未核实 |
| 4 容器镜像 tag 与 digest | 已确认 |
| 5 子模块、下载脚本、运行时拉取 | 已确认 |
| 6 CI Action、密钥、危险触发 | 已确认 |
| 7 恶意包与已知漏洞 | 已确认 |
| 8 主要依赖维护状态 | 已确认 |

## 1. 依赖清单与 lockfile

结论：已确认。Go 与 npm 的锁文件在，并且含间接依赖；镜像里的系统包没有锁文件。

- `backend/go.mod` 第 5–83 行为直接 require 77 条，第 85 行起间接 require 142 条，合计 219。`backend/go.sum` 共 883 行：346 行是模块 zip 的 `h1:`（346 个模块各一个选定版本，这 346 行都有 `h1:`）；537 行是 `/go.mod` 哈希。其中 192 个版本只有 `/go.mod`、没有 zip，不进入构建列表；1 个 zip 版本没有配对的 `/go.mod` 行。219 条 require 的版本都能在 zip 记录里找到。证据：`backend/go.sum:1`，基准 SHA 同上。
- `frontend/pnpm-lock.yaml:1` 为 `lockfileVersion: '9.0'`。`packages:` 下 1122 个包条目（693 个包名，含同一包的多个版本），每条都有 `integrity`。直接依赖的 specifier 仍是范围，锁定版本写在 importers 段，例如 `frontend/pnpm-lock.yaml:15-17` 的 `@apollo/client`。
- 没有 Python／Ruby／Rust 清单。Dockerfile 第 18–25 行与第 145 行的 `apt-get`／`apk add` 不钉包版本，也没有 lockfile。

## 2. 版本钉、校验和、git／URL 直装

结论：已确认。语言锁文件有校验和，也没有 git／URL 直装；构建阶段有一条不钉版本、不带校验和的 `go install`。

- Go：选定版本全部有 `h1:`，见上一节。`backend/go.mod` 无 `replace`、`exclude`、`retract`。
- npm：1122 条都有 `sha512` integrity，抽查 `frontend/pnpm-lock.yaml:2745-2746` 的 `axios@1.18.0`。锁文件里没有 `git+`、`github:`、`link:`、`file:`、自建 tarball。`frontend/pnpm-lock.yaml:7-9` 用 overrides 把 `js-yaml`、`marked` 钉到指定范围或版本。`frontend/package.json` 的 dependencies 仍写 `^`，实际版本以锁文件为准。
- 例外：`Dockerfile:90` 执行 `go install github.com/google/go-licenses@latest`，工具版本不钉、无校验和。README.md 第 614 行的安装包 zip 同样没有校验和（已写入 `audit/supply-chain.md`）。

## 3. 仿冒或可疑包名、安装钩子、非官方源

结论：未核实。根清单和锁文件里能看的部分没有命中；传递依赖的安装脚本不在锁文件里，没有下载包体，不能下结论。

- `frontend/package.json` 的 scripts（第 5–22 行）没有 `preinstall`／`install`／`postinstall`。仓库无 `.npmrc`。pnpm 锁文件没有 `requiresBuild` 字段，因此传递依赖是否带安装脚本未核实。
- 对 693 个 npm 包名的最后一段，与一组常见包名做编辑距离 1 比较，没有命中。这不是完整仿冒包名库，未核对的名字仍未核实。
- Go 模块主机名除 github.com、golang.org、google、gopkg.in、gitlab.com、go.opentelemetry.io、go.uber.org 外，还有 `dario.cat`、`filippo.io`、`go.starlark.net`、`gonum.org`、`gotest.tools`、`howett.net`、`mellium.im`、`modernc.org`、`nhooyr.io`、`pgregory.net`、`sigs.k8s.io`。这些是模块路径里的域名，不是另配的 GOPROXY。域名是否仍由原维护者控制，未逐个核实；已锁定的 zip 由 `h1:` 固定。

## 4. 容器镜像来源、tag、digest

结论：已确认。本仓引用的镜像都没有 digest。多数 tag 是 `latest`、主版本或会浮动的次版本。

镜像表已在 `audit/supply-chain.md`。本次补一条：`.github/workflows/e2e.yml:113` 使用 `mcr.microsoft.com/playwright:v1.61.1-noble`，有 tag，无 digest。安装器目录 `backend/cmd/installer/files/links/` 下的 compose 与根目录对应文件是同一组镜像引用，不是第二套来源。层内容未拉取，层内是否还有问题未核实。

## 5. 子模块、下载脚本、运行时拉取

结论：已确认。没有子模块。仓库内有管道安装文档，运行时会拉容器镜像，并在开关打开时拉 Ollama 模型。

- 无 `.gitmodules`。
- `examples/guides/vllm-qwen35-27b-fp8.md:74` 为 `curl -LsSf https://astral.sh/uv/install.sh | sh`。README 的 wget／curl 见 `audit/supply-chain.md`。构建文件没有执行这些文档命令。
- 运行时拉镜像：`backend/pkg/docker/client.go:1278` 的 `ImagePull`。拉哪一个镜像还受 `backend/pkg/providers/image_selection.go:64` 的空允许列表影响（见既有 F-04）。
- 运行时拉模型：`OLLAMA_SERVER_PULL_MODELS_ENABLED` 默认 false（`backend/pkg/config/config.go:151`）。为 true 时 `backend/pkg/providers/ollama/ollama.go:172` 调用 `client.Pull`。默认路径不会拉。模型注册表内容未核实。

## 6. CI

结论：已确认。Action 都没有钉提交 SHA。没有 `pull_request_target`。存在 `workflow_run`。密钥引用的是 GitHub Secrets，文件里没有写出口令字面量。

- 四个工作流里的 `uses:` 都是主版本 tag（如 `.github/workflows/ci.yml:29` 的 `actions/checkout@v6`）。全库搜索 40 位提交 SHA 的 `uses:`，结果为 0。这些 tag 当前指向哪一次提交，未核实。
- `pull_request_target` 未出现。
- `.github/workflows/e2e-report.yml:7` 使用 `workflow_run`。文件头写明不检出 PR 代码。脚本把产物 `results.json` 里的计数字段转成数字后写入评论（第 119–122 行），不把产物正文当作 shell。这只说明源码写成这样，没有跑过这条工作流，不能写成已经证明无害。
- `.github/workflows/ci.yml:186-190` 的 Docker Hub 登录使用 `secrets.DOCKERHUB_USERNAME` 与 `secrets.DOCKERHUB_TOKEN`，作业条件是 `main` 或 `v*` tag（第 175 行），不在 pull_request 作业里。`.github/workflows/e2e-stand.yml:62-64` 使用环境密钥，并要求标签与 head 仓库等于本仓库（第 29–32 行）。`.github/workflows/e2e.yml:30-31` 权限仅为 `contents: read`。未发现把密钥写进仓库文件。

## 7. 公开漏洞库

查询：`https://api.osv.dev/v1/querybatch`，2026-11-26 11:33–11:34 香港时间。对象是 go.sum 里 346 个带 zip 哈希的选定版本，加上 pnpm 锁文件 1122 个 `包@版本`，合计 1468 次，失败 0。只有 `/go.mod` 哈希、没有 zip 的 192 个 Go 版本未查询。未下载 GHSA 网页正文。

### 依赖是否恶意

结论：未发现问题。上述 1468 个锁版本的返回 ID 里没有 `MAL-`。这只覆盖被查询的版本，不能外推到未查询的 192 个仅 `/go.mod` 版本、镜像层和未下载的包体。

### 已知漏洞

结论：已确认。42 个锁版本有返回，ID 前缀为 GHSA 或 GO，没有 MAL。下表是锁版本，不是「已在本进程里被利用」。

直接依赖里 axios、vite、ollama、`golang.org/x/crypto`、`golang.org/x/net` 的摘要已在 `audit/supply-chain.md`。本次新覆盖的间接与其余锁版本如下。`github.com/golang-jwt/jwt/v5` 不在此表（上次查询该 v5 版本为 0 条）；表中的是另一个模块 `jwt/v4`。

- Go `filippo.io/edwards25519@v1.1.0`：GHSA-fw7p-63qq-7hpr，GO-2026-4503
- Go `github.com/ClickHouse/ch-go@v0.58.2`：GHSA-m454-3xv7-qj85，GO-2025-3603
- Go `github.com/andybalholm/brotli@v1.0.6`：GO-2026-6888
- Go `github.com/buger/jsonparser@v1.1.1`：GHSA-6g7g-w4f8-9c9x，GO-2026-4514
- Go `github.com/docker/cli@v24.0.7+incompatible`：GHSA-p436-gjf2-799p，GO-2026-4610
- Go `github.com/docker/docker@v28.2.2+incompatible`：GHSA-pxq6-2prw-chj9，GHSA-rg2x-37c3-w2rh，GHSA-vp62-88p7-qqf5，GHSA-x4rx-4gw3-53p4，GHSA-x86f-5xw2-fm2r，GO-2025-3830，GO-2026-4883，GO-2026-4887，GO-2026-5617，GO-2026-5668，GO-2026-5746
- Go `github.com/go-jose/go-jose/v4@v4.1.4`：GHSA-4w3g-jfj2-cw2q，GHSA-mgcc-cww5-fp2q，GHSA-vvm8-gf78-28g4
- Go `github.com/go-pg/pg/v10@v10.11.0`：GHSA-6xp3-p59p-q4fj，GO-2025-3764
- Go `github.com/gobwas/ws@v1.0.2`：GO-2026-6940
- Go `github.com/golang-jwt/jwt/v4@v4.5.0`：GHSA-29wx-vh33-7x7r，GHSA-mh63-6h87-95cp，GO-2024-3250，GO-2025-3553
- Go `github.com/gorilla/websocket@v1.5.3`：GO-2026-6278
- Go `github.com/klauspost/compress@v1.18.3`：GHSA-259r-337f-4rfw，GO-2026-5841，GO-2026-6865
- Go `github.com/moby/go-archive@v0.1.0`：GHSA-hfg8-hc9c-6c3h，GO-2026-6253
- Go `github.com/moby/sys/user@v0.4.0`：GHSA-mjcv-p78q-w5fw，GO-2026-6686
- Go `github.com/ollama/ollama@v0.32.5`：GO-2025-3557，GO-2025-3558，GO-2025-3559，GO-2025-3582，GO-2025-3689，GO-2025-3695，GO-2025-3824，GO-2025-4251，GO-2026-5750
- Go `github.com/opencontainers/runc@v1.1.12`：GHSA-9493-h29p-rfm2，GHSA-cgrx-mc8f-2prm，GHSA-jfvp-7x6p-h2pv，GHSA-qw9x-cqr3-wc7r，GHSA-xjvp-4fhw-gc47，GO-2024-3110，GO-2025-4096，GO-2025-4097，GO-2025-4098，GO-2026-5761。这是模块依赖里的 runc 库版本，不是宿主机上的 runc 二进制。
- Go `github.com/uptrace/bun/driver/pgdriver@v1.1.12`：GHSA-h4h6-vccr-44h2，GO-2025-3765
- Go `go.opentelemetry.io/otel@v1.45.0`：GO-2026-6975
- Go `go.opentelemetry.io/otel/exporters/otlp/otlptrace@v1.43.0` 与 `.../otlptracegrpc@v1.39.0`：GHSA-8wmf-6v46-5gfg，GO-2026-6505
- Go `go.opentelemetry.io/otel/sdk@v1.45.0`、`sdk/log@v0.21.0`、`trace@v1.45.0`：GHSA-q7gw-24fr-5p69，GO-2026-6974
- Go `golang.org/x/crypto@v0.55.0`：GO-2026-5932，GO-2026-6354，GO-2026-6355
- Go `golang.org/x/mod@v0.38.0`：GO-2026-6179，GO-2026-6180
- Go `golang.org/x/net@v0.58.0`：GO-2026-6603，GO-2026-6610，GO-2026-6611，GO-2026-6612，GO-2026-6617，GO-2026-6860，GO-2026-6866
- npm `axios@1.18.0`：GHSA-3pq3-5fj3-cg6v，GHSA-44g4-m2mj-wpvx，GHSA-4hqw-qxg8-jxx2，GHSA-542g-h47m-68v8，GHSA-9fr6-4gfg-395g，GHSA-c29m-xwm3-cm6r，GHSA-j8rh-479h-cp32，GHSA-m8m8-qj5v-23w3，GHSA-mghh-pgcx-3jjj，GHSA-r4gj-5m52-g5wh，GHSA-vh66-26gq-q6x8，GHSA-x97p-jq2g-jp4f
- npm `baseline-browser-mapping@2.10.38`：GHSA-w5vr-8v7q-w6rv
- npm `brace-expansion@1.1.15`、`@2.1.1`、`@5.0.6`：GHSA-3jxr-9vmj-r5cp，GHSA-6j4f-fj2g-mc7p，GHSA-mh99-v99m-4gvg，GHSA-q2hr-2g5m-vwhr，GHSA-qhr7-859c-m2p7，GHSA-rgw5-rvv9-x895
- npm `browserslist@4.28.2`：GHSA-73wf-gq98-2v4g，GHSA-c83g-rgw3-j3cx
- npm `follow-redirects@1.16.0`：GHSA-8r9p-f939-6h3c，GHSA-mrwp-f3q8-3g9p，GHSA-q3jh-wrc6-jgq8
- npm `immutable@5.1.6`：GHSA-v56q-mh7h-f735，GHSA-xvcm-6775-5m9r
- npm `minimatch@10.2.5`：GHSA-g3hv-75wg-m249
- npm `postcss-selector-parser@6.0.10`：GHSA-rj75-hqrm-r3gf
- npm `shell-quote@1.8.4`：GHSA-395f-4hp3-45gv，GHSA-pqg4-j6r4-53mv
- npm `source-map-js@1.2.1`：GHSA-68fv-2mgg-jv7q
- npm `undici@7.28.0`：GHSA-2gqq-gqf2-x968，GHSA-2jfj-6hjv-fm6j，GHSA-2m5v-pch6-74hx，GHSA-3wwx-pv8p-q78v，GHSA-3xpg-4rpp-hhhm，GHSA-4cwx-7wf7-3272，GHSA-4qv8-2p9j-2685，GHSA-4w7w-5xrc-9h26，GHSA-55wc-3pg8-jfw6，GHSA-8436-99hf-9mmv，GHSA-8xcm-r25x-g524，GHSA-9fm3-97px-rjf6，GHSA-f72v-6w45-q472，GHSA-jr45-8vmc-qm54，GHSA-m8rv-5g2x-5cg5，GHSA-pmjh-fqhp-jwvm，GHSA-qrxm-fqhp-jwvm，GHSA-r53p-7pc4-xj5r，GHSA-rfgv-xxqx-mfg5，GHSA-rx4f-c7p8-82vq，GHSA-v3r7-h72x-cjcm，GHSA-w293-vg96-wgc3，GHSA-xjj6-f33q-929c
- npm `vite@8.0.16`：GHSA-587h-7wmj-h5pm，GHSA-698v-4r3x-3vcv，GHSA-9jrq-w75r-8gcw，GHSA-r825-w8mm-5pxw，GHSA-rq7h-c2jc-7f22
- npm `vitest@4.1.9`：GHSA-82fw-gwwq-j7x9
- npm `zod@4.4.3`：GHSA-mrhx-c7xc-m3w5

## 8. 主要依赖的维护状态

结论：已确认。多数主要依赖在 2026 年仍有发布；直接依赖 `github.com/jinzhu/gorm` 的模块发布停在 2020 年，锁文件还标了两个已弃用的 npm 包。

查询时间 2026-11-26 11:34 香港时间。模块用 `https://proxy.golang.org/<module>/@latest` 与 `@v/<version>.info`，npm 用 `https://registry.npmjs.org/<name>`，仓库用 `https://api.github.com/repos/<owner>/<repo>`。

- `github.com/jinzhu/gorm`：`@latest` 仍是 v1.9.16，时间 2020-08-13T10:58:45Z。GitHub `jinzhu/gorm` 未归档，最近推送 2024-08-07。替代模块 `gorm.io/gorm` 的 `@latest` 为 v1.31.2（2026-06-22）。本仓锁定的是旧模块，`backend/go.mod:39`。
- 仍在更新的直接依赖（`@latest` 时间，不是本仓锁定版本）：gin v1.12.0（2026-02-28），jwt/v5 v5.3.1（2026-01-28），pgx/v5 v5.13.0（2026-11-11），gqlgen v0.17.98（2026-10-22），ollama v0.40.18（2026-11-24），vxcontrol/cloud v1.2.0（2026-11-17），goose v3.30.2（2026-11-25），moby/client v0.6.3（2026-11-13）。这些仓库的 `archived` 均为 false。
- `github.com/vxcontrol/langchaingo` 锁定版 `v0.1.14-update.8` 的 `.info` 时间为 2026-09-28。代理的 `@latest` 返回 v0.1.14（2025-10-20），因为带后缀的标签不一定被选成 latest。仓库未归档，最近推送 2026-10-13。
- `github.com/vxcontrol/graphiti-go-client` 的 `@latest` 就是本仓的 v0.9.0（2026-03-29）。未归档。之后没有更新的版本，未把它写成弃用。
- npm `@latest`（注册表时间）：axios 1.21.1（2026-11-09），react 19.3.1（2026-10-14），vite 8.3.8（2026-11-11），eslint 10.15.0（2026-11-13），graphql 17.1.0（2026-11-04），@apollo/client 4.3.8（2026-11-09）。均未标 deprecated。本仓锁的版本低于这些 latest。
- 锁文件自带弃用说明：`frontend/pnpm-lock.yaml:3339` 的 `eslint@9.39.4` 写明该版本不再支持；`frontend/pnpm-lock.yaml:5255` 的 `tsconfck@3.1.6` 写明 unmaintained；`frontend/pnpm-lock.yaml:4417` 的 `node-domexception@1.0.0` 建议改用平台自带实现。
