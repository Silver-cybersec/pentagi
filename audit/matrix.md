# 验收矩阵

审计对象：https://github.com/vxcontrol/pentagi 。本环境检出与 2026-11-26 10:28 香港时间 `git ls-remote` 的 `refs/heads/main` 均为 `cffed9485aa1cca70762cd08a57ac62a80406258`。提交时间 2026-10-23 07:20 香港时间。用户给定的基准来源日期是 2026-11-26 09:29 香港时间。

本单启用：代码安全、架构、供应链（含「依赖是否恶意」）、功能与使用、编译与部署要求。后两章全部标「未实测」。未启用且未做：拓扑全表、性能、许可证深挖、私库留存。未运行 pstack，未执行 interrogate。

| ID | 需求 | 检查项 | 证据 | 结论 | 未覆盖 |
|---|---|---|---|---|---|
| R1 | 对用户指定的本仓、本单固定基准 SHA 做只读静态重审 | 记录上游 URL、默认分支、基准 SHA、提交时间（香港时间）；只读源码、清单与锁文件 | 本环境 `git rev-parse HEAD` 与 `git ls-remote https://github.com/vxcontrol/pentagi refs/heads/main`（2026-11-26 10:28 香港时间）均为 `cffed9485aa1cca70762cd08a57ac62a80406258`。作者 Sergey Kozyrenko。`git` 原始提交时间 `2026-10-23T06:20:46+07:00`，即香港时间 2026-10-23 07:20。默认分支 main。可写远程是 fork https://github.com/Silver-cybersec/pentagi ，审计对象仍是上游该 SHA | 已在该树上阅读。未改产品代码 | `refs/heads/master` 在同一次 ls-remote 中无返回。README 引用的 master compose URL 未下载正文 |
| R2 | 代码安全 | 故意漏洞与对部署者／宿主机的真实风险分开；危险执行、默认凭据、硬编码密钥、数据外传、隔离、暴露面、容器／宿主挂载；高／中／低 | `audit/findings.json`：高 F-01、F-02、F-03；中 F-04 至 F-09；低 F-10 至 F-13。每条有路径、行号和上游 SHA 链接 | 未发现靶场关卡式故意漏洞。真实风险集中在宿主 Docker 套接字、公开默认管理员口令可跳过改密、worker 端口默认 0.0.0.0 | 未动态验证逃逸、未登录试默认口令、未逐个 GraphQL resolver 做越权、未确认 worker 能否从 bridge 访问宿主回环端口 |
| R3 | 架构（含与学习目标相称的功能阅读） | 组件、信任边界、数据与权限流；关键流程能否被绕过的已读证据 | `backend/cmd/pentagi/main.go`、`backend/pkg/server/router.go`、`backend/pkg/docker/client.go`、`backend/pkg/graph/context.go`、`backend/pkg/controller/flows.go`。宣称对照全表在主报告「功能与使用」章 | 控制面、数据库、worker、外部模型是四条信任边界。GraphQL 对 flow 有归属检查；默认管理员会话在改密前已经完整可用 | 未读完每一个 specialist agent 的提示词分支；订阅以外的每个 resolver 未逐个核对 |
| R4 | 供应链，且必须单列「依赖是否恶意」 | 包与锁文件、镜像 tag/digest、子模块、管道脚本、校验和、仿冒名、安装钩子、非官方源、OSV／GHSA | `audit/supply-chain.md`。OSV 查询 2026-11-26 10:30 香港时间。无子模块、无 go replace、前端无安装钩子 | 抽查无 MAL 记录。axios、x/net、ollama、vite、x/crypto 有普通漏洞公告。镜像无 digest。未把未拉取的层写成安全 | go.sum 与 pnpm-lock 传递依赖未全量查询；镜像层未拉取；vxcontrol 三个 Go 模块源码不在本仓 |
| R5 | 产出精练中文报告 | 四个交付文件；每节先结论再短分条 | `audit/RERUN-20261126.md`、`audit/matrix.md`、`audit/findings.json`、`audit/supply-chain.md` | 已写。未改总报告、飞书或 Page | 无单独 topology.md、无运行截图、无 pstack 输出 |
| R6 | 查是否存在可对照的旧仓库安全审计基准 | 本课题既有审计记录 | 任务说明写明总报告对该仓为未审计。本仓 `audit/` 在该 SHA 上原本不存在 | 无旧仓库安全审计基准。不写差异章 | 无。这不是缺口 |
| R7 | 不新建私库、不改产品代码、不把静态阅读写成已运行验证 | 交付物与表述 | 分支 `cursor/rerun-20261126-pentagi` 只新增 `audit/`。PR 指向 fork 的 main，不合并，不对上游开 PR | 遵守。功能与部署两章每条标未实测 | — |
| R8 | 功能与使用（用户 2026-11-26 10:14 香港时间同意启用） | README／文档宣称对照实现；安装后入口、关键配置、一条典型工作流 | 主报告对应章。宣称来自 README.md 第 71–90 行与第 1059–1141 行；实现位置见该表 | 多数主路径在源码中找得到。隔离、水平扩展、20+ 工具、API token 在空盐下为部分或与实现不一致。全部未实测 | 未点击 UI，未创建 flow，未核对每个提示词工具是否真的装在 kali 镜像里 |
| R9 | 编译与部署要求（同上启用） | Dockerfile、compose、包管理器、语言与数据库版本、硬件、外网、文档与代码不符 | 主报告对应章。Dockerfile、四份 compose、`backend/go.mod`、`frontend/package.json`、`frontend/.nvmrc`、README 第 588–594 行 | 构建入口在仓库里。版本与默认值多处文档和代码不一致。未编译、未安装、未起容器，不得称为已验证可部署 | 未执行 `docker build`、`go test`、`pnpm install`；未测量内存；GPU 需求只存在于可选 vLLM 指南 |
