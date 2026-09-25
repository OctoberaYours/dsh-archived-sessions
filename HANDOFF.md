# dsh-archived-sessions 维护文档（接手必读）

> **这是独立维护的 fork**，上游 `Zephyr-vibe/dsh-archived-sessions` 已停更（最后 push 2026-08-21）。
> 本仓库从 `0.2.0` 起独立演进，目标是跟随 DSH 核心当前版本 —— 目前适配 **DSH `0.1.7-rc.2`**。
> 上游 `v0.1.5` 面向 `0.1.0-rc.8`，其代码在 rc.2 上会**静默失效**（详见 README 更新日志）。

## 项目概况

DSH Web 插件：设置页的「会话管理」。双端结构：

- **host 端**：`lib/index.js`（Node ESM，插件主逻辑：HTTP API、删除/归档、详情构建）
- **client 端**：`lib/client.js`（浏览器 bundle，设置页 UI）

## 生效机制（重要）

| 改动 | 生效方式 |
|---|---|
| `lib/client.js` | 浏览器刷新页面即生效（bundle URL 带内容哈希 rev） |
| `lib/index.js`（host） | **必须重启 DSH**（进程内 ESM 模块缓存按 URL 缓存，不会热更） |
| `package.json` | **必须重启 DSH**（profile 组合在启动时读取） |

⚠️ 重启会中断用户正在进行的对话/任务。**重启前先确认**，不要擅自重启。

## 本机开发环境（link 安装）

本仓库以 **junction + `link:` 依赖**方式挂进 profile，改代码即生效：

```
C:\Users\<你>\.dsh\profiles\desktop\node_modules\dsh-archived-sessions
    →(junction)→ C:\Users\<你>\Code\dsh-archived-sessions
```

profile 的 `package.json` **两处都要写**（只写一处会被静默启停或解析不到入口）：

```jsonc
{
  "dependencies": { "dsh-archived-sessions": "link:C:/Users/<你>/Code/dsh-archived-sessions" },
  "dsh": { "profile": { "bundles": [ /* … */ "dsh-archived-sessions" ] } }
}
```

> 注意：DSH **Desktop** 用 `~/.dsh/profiles/desktop`，CLI web 用 `~/.dsh/profiles/web`。
> Desktop 的 profile 由 Electron 独占管理，CLI 拒绝操作（`error: profile "desktop" is managed exclusively by the Electron application`）。

## 架构与关键实现

### host（index.js）

- `apply(ctx)`：注册 `/archived/api/*` 路由（loopback 围栏 + POST + 方法白名单 `ARCHIVED_API_METHODS`）
- API：`details`（详情+统计+lineage）、`delete`（支持 subagentIds/filePaths 细粒度）、`delete-file`、`open-folder`、`archive`、`unarchive`
- `buildDetails`：从会话事件构建统计与文件列表；**files 列表做 stat 过滤**；lineage 含 `children`（分叉）和 `subagents`（子代理）
- `deleteSession(ctx, id, { cascade, deleteFiles, subagentIds, filePaths })`：
  - 子代理删除：显式 `subagentIds` > `cascade`（`collectDescendants` 递归收集全部后代）
  - 文件删除：`filePaths` 指定时逐个删（**工作区围栏校验** + 只删普通文件）+ 删记录 log；`deleteFiles:false` 只删记录；默认删整个会话目录
  - 文件/文件夹删除后 `pruneEmptyDirs` **向上清理空目录**（边界 = 工作区根集合，根绝不删）
- 状态码约定：404 session-not-found / 400 校验错误 / 403 越界与非法目标 / 405 非 POST / 409 运行中 / 501 缺原语
- 注册的路由经 `ctx.effect` 清理（无监听器泄漏）

### client（client.js）

- `ArchivedSessionsSection`：主组件（useSessions 拆订阅 byId/current/phase 防重渲染）
- `SessionRow`：memo 行组件（props 全基本类型/稳定引用）
- `normId`：id 归一化（剥离 `session-` 前缀）—— **byId 的 key 与 parentId 格式可能不一致**，所有归属匹配必须双向 normId
- `openDeleteConfirm`：收集选中会话的**全部后代子代理**（递归）+ 全部文件（含子代理产出），推导可删文件夹（**排除工作区根**）
- `refreshSessionLists`：删除/归档后刷新会话与工作区列表。**注意 cordis 的 inject 访问门禁**（见下）
- 路径工具：`dirOf`（父目录）、`baseName`（文件名）—— client 无 node path，需自实现

## 0.1.7-rc.2 适配要点（踩过的坑，别再踩）

### 1) cordis 4 的 `inject` 是**访问门禁**，不只是依赖声明

未声明的服务用 `ctx.<name>` 读取会**直接抛错**：

```
Error: cannot get property "sessions" without inject
```

`ctx.get("<name>")` 才是无门禁的宽容读取。若把 `ctx.<name>` 包在 `try{}catch{}` 里"best effort"，
异常会被静默吞掉 —— 表现为"功能莫名不工作，且毫无报错"。

### 2) 服务注册有两种写法，别只扫一种

| 服务 | 注册方式 |
|---|---|
| `slots` | `@deepseek-ai/dsh-client-ui-renderer` — `super(ctx,"slots")` |
| `sessions` | `@deepseek-ai/dsh-session` — `super(ctx,"sessions")` |
| `workspaces` | `@deepseek-ai/dsh-api-workspace-controller` — `super(ctx,"workspaces")` |
| `locale` | `@deepseek-ai/dsh-client-locale` — **`ctx.provide("locale", …)`** |

用"只匹配 `super(ctx,…)`"的脚本去扫会漏掉 `ctx.provide(...)`，从而得出错误结论。

### 3) `dsh.client.inject` 是**模块图边**，不是服务名清单

合法取值只有两类：

- **包行**：该包有 `lib/client.js`（例：`@deepseek-ai/dsh-client-locale`）→ 会加图边，保证排在消费者前面
- **static-table（seed）名字**：由 shell 直接供给，无图边。rc.2 的 seed 表是固定的 9 项：
  `react` / `react/jsx-runtime` / `react-dom` / `react-dom/client` / `@deepseek-ai/cordis` /
  `@deepseek-ai/dsh-client-store` / `@deepseek-ai/dsh-client-ui-slots` /
  `@deepseek-ai/dsh-client-ui-primitives` / `@deepseek-ai/dsh-client-ui-dockkit`

既非包行也非 seed → 无法解析。**官方插件只列包行**，seed 名写了是合法 no-op 但无用。

> `require("react")` 能成功，是因为 react 在 seed 表里，与 `dsh.client.inject` 无关。

### 4) `sessionPersistence` 契约（rc.2 重构）

| rc.1 及更早 | rc.2 |
|---|---|
| `inspect(id)` | `open(id,"read")` → `handle.read()` |
| `readRaw(id)` | 移除 |
| `remove(id)` | 移除 |
| `artifactInfo(id)` | `stat(id)` → `{ header, revision, sizeBytes }` |
| `list()` → `[header, …]` | `list()` → `[{ header, revision, sizeBytes }, …]` |
| `coordinator.retirements` | 移除 |

**危险点**：旧代码对这些调用都有 `typeof … === "function"` 守卫，失配后不报错、只是永远拿不到数据 ——
包括 `remove()` 消失后 `removeLog()` 静默跳过，变成"**报告删除成功但文件还在**"。
因此所有删除分支都必须显式校验目录可定位，不可依赖守卫。

### 5) live Session 没有公开 `events`

事件日志是私有 `log`，公开读取须走 `snapshotEvents()`（返回冻结快照）。
`[...live.events]` 必抛 `TypeError: live.events is not iterable`。
官方同款用法见 `@deepseek-ai/dsh-session-query` 的 `snapshotLive()`。

### 6) `decodeStorageRecord` 不再从 `@deepseek-ai/dsh-session` 导出

函数本体仍在 `lib/types/chunk-rows.js`，但该包 `exports` 白名单不含 `chunk-rows`，无法导入。
**它不是死代码**：分块存储行占会话日志约 27%，删掉会让这些事件整批丢失。
本仓库已按官方实现逐字节等价内联到 `lib/index.js` 顶部。

## 版本与发布流程

1. 改代码 → `node --check lib/index.js lib/client.js`
2. 升版本：`package.json` 的 `version` + README changelog（**中英两区都要改**）
3. 校验：`npm pack --dry-run` 确认打包内容
4. 提交（署名 `OctoberaYours <70874019+OctoberaYours@users.noreply.github.com>`）→ 推送到 `fork` remote
5. README 中英切换用单文件锚点（`[中文](#中文) | [English](#english)`），中文在前

> GitHub 推送若失败（`schannel: failed to receive handshake`），是代理只配了 `http.proxy` 没配 `https.proxy`：
> `git -c http.proxy=http://127.0.0.1:10808 -c https.proxy=http://127.0.0.1:10808 push fork <branch>`

## 测试方法

- **API 层**：`Invoke-WebRequest` 直连 `http://127.0.0.1:<port>/archived/api/*`（POST + JSON body），断言状态码与响应
  - 判别路由是否注册：405（路由在，方法不对）vs 404（路由不在）
- **删除行为**：用临时测试会话/文件，删除后文件系统复核（文件没了、空目录清了、根还在）
- **危险操作前先造副本**：验证删除逻辑时用 `DSH_HOME` 指向会话树副本，避免动到真实数据
- **UI 层**：生成测试素材（子代理链 + 分层文件夹），浏览器手动验证

## 常见坑（务必注意）

1. **id 格式混用**：`session-` 前缀 vs 纯 uuid —— 任何 parentId 比较都要 `normId` 双向
2. **files 是事件记录**：不是磁盘扫描；删除后要 stat 过滤，且删除弹窗重新拉详情时才一致
3. **工作区根保护**：文件夹删除/空目录清理的边界都是工作区根集合，根绝不删
4. **`pruneEmptyDirs` 边界**：只认工作区根集合（stopSet），不要用 `sessionsRoot`（工作区文件不在会话目录下）
5. **不要擅自重启 DSH**：先确认用户没有正在跑的任务
6. **删除弹窗默认不勾选**：用户明确要求两个选项默认关闭
7. **emoji/间距**：文件夹行必须用 `label.selectAll` 同款组件（历史教训：自定义样式导致行高/间距差异）
8. **插件的 on-disk 布局与官方一致**：`$DSH_HOME/sessions/<projectKey(cwd)>/<encodeSegment(id)>/`，
   删除逻辑依赖它定位目录（已与官方 `projectDir` 逐字段核对）

## 遗留事项

- steps 统计依赖"step 编号全局递增"约定（有注释），若官方改按 turn 重置需改为事件计数
- `snapshotEvents()` 在 rc.2 已标记 `@deprecated`（官方在去同步化事件读取），未来版本可能移除，
  届时需迁移到异步读取路径
- `schemastery` 以**裸包名**导入（`import z from "schemastery"`），而 rc.2 宿主里是 `@deepseek-ai/schemastery`。
  目前靠本仓库自带的 `node_modules/schemastery`（3.18.0）解析成功，与宿主的 3.18.4 存在版本偏斜 —— 潜在风险
- `USAGE.md`、本文件是独立文档（README 未引用）
