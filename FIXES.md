# 本地修复：DSH 核心 0.1.3-alpha.2 的 persistence 契约变更

上游：<https://github.com/Zephyr-vibe/dsh-archived-sessions>（基线 0.1.5，`lib/` 逐字节一致，仅换行符不同）
本 fork 版本：`0.1.5-alpha2-compat.1`

## 症状

设置 →「会话管理」→ 删除会话：

```
成功 0 项，失败 32 项：找不到该会话的记录（会话不存在）
```

同一根因还让「归档 / 取消归档 / 打开记录目录」全部报同一句 404，详情面板报 500。

## 根因

`SessionPersistence`（`ctx.sessionPersistence`）的读取契约在两代核心之间换形：

| | 0.1.0-rc.8（插件写作目标） | 0.1.3-alpha.2（当前宿主） |
|---|---|---|
| `list()` | `SessionHeader[]`（扁平，带 `.id`） | `SessionPersistenceSnapshot[]`（header 在 `.header`） |
| `listSnapshots()` | 有 | 无 |
| 读整包日志 | `inspect(id, signal)` / `load(id)` | `open(id,'read')` → `handle.read()` → `handle.close()` |
| 物理体积 | `artifactInfo(id)` | `stat(id).sizeBytes` |
| 原始工件 | `readRaw(id)` | 无 |
| 退休协调 | `coordinator.retirements` | 无 |
| `locate(meta)` | 公开 API | 降级为拒绝诊断钩子（private） |

运行时的铁证来自核心自身——`@deepseek-ai/dsh-workspace@0.1.3-alpha.2` `lib/index.js:730`：

```js
return (await this.ctx.sessionPersistence.list()).map((snapshot) => snapshot.header);
```

插件仍按 rc.8 把每一项当扁平 header 用（`meta.id === sessionId`），于是恒不匹配 →
`findSessionMeta()` 恒返回 `undefined` → `deleteSessionSingle()` 抛 404。

## 修复策略：形状归一化，而不是硬切新 API

发布的 tarball 只有编译后的 `lib/`，宿主可能是 rc.8 也可能是 alpha.2，所以全部改
成「两代契约都成立」的读法，新增 `toSessionHeader()` / `listStoredHeaders()` /
`readStoredSession()` / `isNotFoundLike()` / `sessionNotFoundError()` 五个内部辅助：

1. `list()` 每项先试 `.header` 再退回自身，只接受 `id` 为字符串的项；有
   `listSnapshots` 时优先用它。
2. 整包日志：`inspect` → `load` → `open('read')+read()+close()`（`finally` 里必关句柄）；
   核心的 `*NotFoundError` 归一成插件自己的 404，其余原样抛出（不把真故障伪装成不存在）。
3. `artifactInfo` 缺失时体积改取 `stat().sizeBytes`。
4. `locate()` 不可用时，用本模块已有的官方布局镜像 `sessionDirFor(meta)` 兜底推导记录
   目录，且仅在该目录确实存在、且严格位于 `sessionsRoot` 内时才采用；删 log 时退化为
   只清目录内的记录文件，目录本身交给后续统一清理。记录文件的匹配式按**盘上实测命名**
   收紧：真实核心的代次文件是 `session.jsonl.zstd` 与 `session.v2.jsonl.zstd`，只判
   `\.jsonl$` 或 `\.zst$` 两条都不中（`.zstd` 不以 `.zst` 结尾），故改为
   `^session[^/]*\.(jsonl|zst|zstd|gz|br|lz4)$`。
5. `coordinator.retirements` 不存在 → 删除后的空目录清理改为有界重试（4 次、25/50/75ms
   退避），仍能压住 retire 尾部 flush 重建目录的竞态。
6. `import { decodeStorageRecord } from "@deepseek-ai/dsh-session"` 改为惰性可选解析。
   该具名导出在新核心里已经不存在，而**静态具名导入缺导出会让整个模块在加载期就崩**——
   哪天依赖解析漂到 alpha.2 那份 dsh-session，「会话管理」会整块消失，而不是只坏一条
   fallback。
7. **live 会话的详情**：alpha.2 的 `Session` 已经没有 `.events` 属性（公开替代品是
   `snapshotEvents()`），原码 `[...live.events]` 抛
   `live.events is not iterable` → 对**当前打开的会话**点详情必然 500。改为
   `snapshotEvents()` 优先、退回 `.events`、两者皆无则空数组（不再 500）。

   这条是第 7 点值得单独记：它是**真机批量删除之后**跑审计时才暴露的，此前 9 条 mock
   断言全绿也照样漏着——因为 mock 一律只喂「非 live」形态。教训：live / 非 live 是两条
   独立读取路径，覆盖率必须按分支算，不能按 API 名字算。

## 顺带修掉的破坏性缺陷

归档集孤儿清理那处（原 `existing.add(h.id)`）加进去的是 `undefined`，
`filter` 会把**所有非 live 的归档会话 id 从 `archivedSessionIds` 里静默抹掉**——
不是报错，是丢数据。现在走同一个归一化列表，回归测试里「非本次目标的 archived id
必须保留」那条断言专门钉住它。

## 复现与验证

补丁与测试脚本在同级的 `../dsh-session-manager-fix/`：

```bash
node patch-alpha2-compat.mjs lib/index.js --check   # 只校验锚点，不写盘
node patch-alpha2-compat.mjs lib/index.js           # 幂等打补丁（锚点数不符即拒写）
node run-tests.mjs lib/index.js                     # 11 项行为断言
```

红→绿：对未修版跑同一套测试得 **1/11**（失败信息逐字复现用户看到的
`找不到该会话的记录（会话不存在） status 404`，以及 live 分支的
`live.events is not iterable`），修后 **11/11**；其中 rc.8 扁平形状那两条断言
修前修后语义一致，证明没有把旧契约的路径改坏。

## 真机验收（宿主 0.1.3-alpha.2，端口 3080）

修复前，同一台机器上的真实失败：

```
POST /archived/api/details  → {"code":"internal","message":"persistence.inspect is not a function"}
POST /archived/api/archive  → HTTP 404 {"code":"session-not-found","message":"找不到该会话的记录（会话不存在）"}
（用户批量删除后才暴露）details 一个 live 会话 → HTTP 500 {"message":"live.events is not iterable"}
```

修复并重启宿主后（插件 entry 已指向本 fork）：

| 调用 | 结果 |
|---|---|
| `details` 一个仅持久化的会话 | HTTP 200，`sizeBytes` 有值（证明 `stat()` 体积回退生效） |
| `archive` 同一会话 | HTTP 200，`archivedSessionIds` 写入该 id |
| `unarchive` 复原 | HTTP 200，归档集回到 `[]`（无残留副作用） |
| `delete` 一个一次性的子代理会话产物 | HTTP 200，目录消失，会话目录数 35→34，其余 34 条零误伤，归档集未受污染 |
| `details` 当前打开的 live 会话（补丁 7 之后复测） | HTTP 200，`turns=6 steps=139`（此前 500） |

## 用户真实批量删除后的状态审计

用户在 UI 里把原先失败的 32 条删除后复查：

| 检查 | 结果 |
|---|---|
| 剩余会话目录 | 2（34 → 2，与删除条数吻合） |
| 空目录残留 | 0 |
| 只剩壳 / 无 log 文件的会话目录 | 0 |
| `archivedSessionIds` 被连带破坏 | 否（保持 `[]`，与删除前一致） |
| 删除后服务仍可用 | 两条会话的 `details` 均 HTTP 200 |

唯一遗留（**非本次回归，未擅自扩大删除语义**）：项目级容器目录
`sessions/--C-Windows-System32--`、`sessions/--E-zcode--` 在其下会话清空后成为空目录。
插件的清理边界只到会话目录，工作区文件清理的边界才是工作区根；核心下次在这两个项目
下建会话会自动 `mkdir`，不影响功能。要一并回收需另加“删完会话后回收空项目目录”的逻辑。

