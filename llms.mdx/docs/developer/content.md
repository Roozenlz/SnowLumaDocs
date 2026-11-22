# 开发者文档 (/zh/docs/developer)



SnowLuma 是 TypeScript + 原生模块混合项目。TypeScript 负责运行时编排、OneBot 实现、协议桥接、SDK 和 WebUI；原生模块负责 Hook、注入和高性能 WebSocket 能力。

## 环境要求 [#环境要求]

| 项       | 要求                       | 依据                                        |
| ------- | ------------------------ | ----------------------------------------- |
| Node.js | `^24.22.1` 或 `>=26.12.1` | `package.json` 的 `engines`；低于此版本的安全基线不受支持 |
| pnpm    | `11.25.0`（固定版本）          | `packageManager` 字段                       |
| QQ      | NTQQ 有界面客户端              | Hook 需要可见窗口                               |

`.node-version` 固定为 `24.22.1`，CI 与发行包都按它构建。只改 `.node-version` 不会升级已有安装的运行时。

## Monorepo 结构 [#monorepo-结构]

pnpm workspace，14 个包：

```text
SnowLuma/
├─ packages/
│  ├─ core/        @snowluma/core       应用入口、Bridge 编排、WebUI 后端、发行装配
│  ├─ bridge/      @snowluma/bridge     QQ 进程发现、登录识别、Hook 会话
│  ├─ protocol/    @snowluma/protocol   封包解码、类型化 QQ 事件、身份解析
│  ├─ onebot/      @snowluma/onebot     动作校验与分发、事件转换、HTTP / WS 适配器
│  ├─ common/      @snowluma/common     日志、配置、版本与通用工具
│  ├─ sqlite/      @snowluma/sqlite     异步 SQLite 门面、原生加载、预编译选择
│  ├─ proto-defs/  @snowluma/proto-defs 类型化协议 schema
│  ├─ proton/      @snowluma/proton     编译期 protobuf 编解码与构建集成
│  ├─ websocket/   @snowluma/websocket  本仓 WebSocket 实现（原生加速）
│  ├─ runtime/     @snowluma/runtime    发行清单、启动器、随附原生件
│  ├─ webui/       webui                React + Vite 控制台
│  ├─ ui/          @snowluma/ui         对外发布的 UI 组件包
│  ├─ sdk/         @snowluma/sdk        第三方 TypeScript OneBot 客户端
│  └─ mcp/         @snowluma/mcp        MCP server 与生成的 action 目录
├─ docker/         容器构建材料（Dockerfile / start.sh / supervisord / compose）
├─ tools/          辅助脚本（版本 bump、release、发行布局校验）
├─ compat/         兼容层
├─ assets/         README 与文档用图
└─ package.json
```

原生件放在 `packages/runtime/native/`，由构建拷贝到 `dist/native/`。

## 核心启动链路 [#核心启动链路]

入口在 [`packages/core/src/index.ts`](https://github.com/Roozenlz/SnowLuma/blob/main/packages/core/src/index.ts)：

```text
loadRuntimeConfig()           ← 读 config/runtime.json / env
  ↓
applyPendingUpdate()          ← 启动前先落地已暂存的更新
  ↓
HookManager(autoLoadOnDiscovery)
  ↓
BridgeManager
  ↓
OneBotManager                 ← 按 UIN 拉起 OneBotInstance
  ↓
initWebUI(port=webuiPort)     ← 仅完整构建启用
```

启动时创建：

* **`HookManager`** —— 发现 QQ 主进程并注入 native addon；`hookAutoLoad: true` 时每发现一个新 PID 自动注入。
* **`BridgeManager`** —— 把命名管道里的 Hook 消息解码成桥接事件；缓存 rkey / 联系人 / 群成员。
* **`OneBotManager`** —— 按 UIN 创建 `OneBotInstance`；持有 `MessageStore` / `MediaStore` / `NetworkManager`。
* **`initWebUI`** —— 完整构建中启动 WebUI，端口默认 `5099`（`runtime.json.webuiPort` 可改）。

Core 保持单一 bundle，原生库惰性加载，这样启动时能先完成被中断更新的恢复，再打开原生文件。

## Hook 与原生模块 [#hook-与原生模块]

Hook 原生模块通过 `process.dlopen()` 加载，源码在独立的 [`SnowLuma/nnphook`](https://github.com/SnowLuma/nnphook) 仓库。

平台命名约定：

```text
snowluma-win32-x64.node    snowluma-win32-x64.dll
snowluma-linux-x64.node    snowluma-linux-x64.so
snowluma-linux-arm64.node  snowluma-linux-arm64.so
```

WebSocket 原生件同规则：`websocket-<platform>-<arch>.node`。

加载搜索路径：

```text
native/
dist/native/
packages/runtime/native/
```

原生 addon 暴露：

* **`getAllMainProcess`** —— 枚举 QQ 主进程。
* **`loadModuleManual`** —— 向目标进程手动映射注入模块。
* **`unloadModuleManual`** —— 卸载已注入模块。

Windows 默认目标进程名 `QQ.exe`；Linux 是 `qq`。接口已全面异步化，应用与原生产物必须成套更新。

## OneBot 生命周期 [#onebot-生命周期]

桥接层检测到 QQ 会话启动时：

1. 加载 `config/onebot_<uin>.json`（缺则从全局复制）。
2. 构造 `OneBotInstance`。
3. 起 HTTP / WS / 反向 WS / HTTP POST 适配器（按 `networks.*` 配置）。
4. 异步预热好友列表、群列表、群成员列表。
5. 桥接事件 → OneBot 事件转发。

会话关闭时对应实例释放。

## WebSocket 包 [#websocket-包]

`@snowluma/websocket` 是项目自带的 WebSocket 实现，接口尽量贴近 `ws`，但带原生加速。

导出：

* **`WebSocket`** —— 客户端构造器。
* **`WebSocketServer`** —— 服务端构造器。
* **`Server`** —— `WebSocketServer` 别名。
* **`InternalWebSocket`** —— 内部连接实现。

依赖平台原生 `.node`，同样从 `native/` 搜索路径加载。

## SDK 与 MCP [#sdk-与-mcp]

`@snowluma/sdk` 是面向第三方的 OneBot 客户端，类型完备：

* HTTP / WebSocket / 反向 WS 三种传输。
* 自动重连、事件订阅、ack 关联。
* 与 SnowLuma 核心共享 action 类型，避免漂移。

`@snowluma/mcp` 把 action 目录暴露成 MCP 工具，供 agent 调用。两者都可独立发布到 npm，不依赖 SnowLuma 运行时。

## 开发命令 [#开发命令]

```bash
pnpm install --frozen-lockfile   # 锁文件不匹配时先解决依赖变更，不要随手重写
pnpm dev              # core 开发模式
pnpm dev:web          # WebUI 开发服务器
pnpm build            # 仅 core
pnpm build:webui      # 仅 WebUI
pnpm build:sdk        # 仅 SDK
pnpm build:mcp        # 仅 MCP
pnpm build:all        # core + SDK + MCP + UI + WebUI
pnpm test             # workflow 策略检查 + 全 workspace 测试
pnpm test:workflows   # 仅 workflow 策略测试，不写 GitHub
pnpm test:native      # 原生冒烟测试（需要匹配的原生件）
pnpm typecheck        # 全仓类型检查
pnpm lint             # 全仓 lint
pnpm ci               # 本地复现 CI 的全部命令
```

## 构建顺序 [#构建顺序]

完整发行版：

```bash
pnpm build:all
```

依次执行：

1. `@snowluma/core` 完整构建（`BUILD_WEBUI=true`）
2. `@snowluma/sdk`
3. `@snowluma/mcp`
4. `webui`（`@snowluma/ui` 先于它构建）

产物统一输出到 `dist/`，随后用 `pnpm check:release-layout` 校验清单与原生件完整性。

## 版本与发布 [#版本与发布]

版本号统一来自根 `package.json`，各包通过 `@snowluma/common/version` 读取。

```bash
pnpm bump 2.4.3        # 同步根与所有 workspace 包的 version
pnpm release 2.4.3     # 完整发布：CI → bump → dev → Promote → main → tag
pnpm release 2.4.3 --dry-run   # 只打印计划
```

`pnpm release` 依赖 `dev` 分支上的 `chore(release): vX.Y.Z` 提交触发 Promote 工作流；发布链路的中途失败可用同一个命令重跑续做，不要 amend 发布提交。

只维护自己的 fork、不跑 Promote 时，手动同步版本号并打 tag 即可：

```bash
# 改完全部 package.json 的 version 后
git commit -am "chore(release): v2.4.3"
git tag -a v2.4.3 -m "v2.4.3"
git push origin main v2.4.3
```

tag 触发 `release.yml`：三平台构建 → 发布 GitHub Release → 构建并推送 Docker 镜像。

## CI 工作流 [#ci-工作流]

| 工作流                           | 触发        | 作用                          |
| ----------------------------- | --------- | --------------------------- |
| `ci.yml`                      | PR / push | test、typecheck、build、lint   |
| `dev-build.yml`               | 任意分支 push | 三平台构建；`dev` 分支额外发布 `dev` 镜像 |
| `release.yml`                 | `v*` tag  | 发版 + 推送 Docker 镜像           |
| `docker-image.yml`            | 被上面两者调用   | 可复用的镜像构建实现                  |
| `dispatch-docker-release.yml` | 手动        | 重建历史版本的镜像，不用重打 tag          |

镜像构建需要的仓库配置：

| 类型       | 名称                   | 说明                          |
| -------- | -------------------- | --------------------------- |
| Secret   | `DOCKERHUB_USERNAME` | Docker Hub 账号               |
| Secret   | `DOCKERHUB_TOKEN`    | Access Token（Read & Write）  |
| Variable | `DOCKER_IMAGE`       | 镜像名，如 `roozen/snowluma`     |
| Variable | `DOCKER_PLATFORMS`   | 如 `linux/amd64,linux/arm64` |

## 文档站 [#文档站]

本站在独立仓库 [`Roozenlz/SnowLumaDocs`](https://github.com/Roozenlz/SnowLumaDocs)。

技术栈：Next.js 16 + Fumadocs + Tailwind 4 + MDX。推 `main` 即由 `.github/workflows/deploy.yml` 静态导出到 `gh-pages`。

OneBot API 参考由文档仓的 `tools/generate-onebot-openapi.mjs` 从主仓的 action 定义静态扫描产生，落到 `content/api/openapi.json`。Action 增删后重新生成并提交。

## 贡献注意事项 [#贡献注意事项]

* **不要绕过 BridgeManager**：OneBot 层一律通过 `BridgeInterface` 调用 NTQQ 能力，便于解耦和测试。
* **保持配置可自动生成**：新增字段需向后兼容；解析时归一化、写回磁盘。
* **原生模块按平台分发**：新加平台时同步 `.node` 与可注入库。
* **优先补测试**：核心 OneBot 行为通过 `vitest` 验证，参考各包的 `tests/`。
* **改 action 要重生成目录**：改完后跑 `pnpm --filter @snowluma/mcp gen`，并把生成的 `catalog.ts` / `catalog.json` 一并提交；CI 的 `tools/check-action-catalog.mjs` 会拒绝漂移。
* **格式化**：TypeScript 走四空格缩进与 `pnpm format`，不要在业务代码里手调风格。
