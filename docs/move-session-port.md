# 迁移会话（move session）移植记录

> 日期：**2026-10-05**。目标宿主：**DSH `0.2.0-rc.2`**。
> 上游参考：[hucj09/dsh-move-session](https://github.com/hucj09/dsh-move-session) `v0.1.3`（针对 0.1.x 编写）。
> 移植去向：本插件 `lib/index.js`（宿主端 `moveSessionToWorkspace`）+ `lib/client.js`
> （`MoveSessionAction`，注册进 `conversation.session.header.actions`）。

---

## 一、结论先行

上游的设计与调用链在 `0.2.0-rc.2` 上**基本可用**，但有一处方法确实消失、若干处行为需要
重新确认。本插件没有逐字照抄，而是先读取宿主的真实源码逐条核对，再用**真实会话日志**
跑真实校验。

核对过程中我**下过一个错误结论并公开更正**，见第四节 —— 这一节比结论本身更值得保留。

---

## 二、逐项核对（全部为源码实证，非推断）

核对方式：按 Chrome-pickle 头部解析 `E:\DeepSeek Harness\resources\app.asar`，
直接读取目标包源码，并用 DSH 自带 node（`resources\runtime\bin\node.cmd`）执行。

| 上游依赖 | rc.2 状态 | 处置 |
| --- | --- | --- |
| `workspaceRegistry.get/list/attachSession/archiveSession/requireState/setState` | 全部存在 | 直接用 |
| `agents.get/create` | 全部存在 | 直接用 |
| `sessions.get/flush` | 全部存在 | 直接用 |
| `agentPresets.resolve/mount` | 存在（包已更名为 `dsh-agent-preset-registry`，服务名仍是 `agentPresets`） | 直接用 |
| `sessionTitle.rename` | 存在 | 直接用 |
| `sessionQuery.readTitleSnapshots` | 存在 | 直接用 |
| **`sessionPersistence.readFrom`** | **不存在** | 改用本插件既有的 `readStoredSession()` |
| `persistence.inspect/open/stat/list` | 存在（rc.2 是 `open()` + handle `read()` 契约） | 经 `readStoredSession()` 兼容两条路径 |

### 唯一真实缺失：`persistence.readFrom`

上游第 2 步是：

```js
read = await persistence.readFrom(sessionId, 0)
```

rc.2 的 `session-persistence-jsonl` 上没有 `readFrom`（实测枚举该类方法确认）。rc.2 的契约是：

```js
handle = await persistence.open(sessionId, "read", { signal })
const read = await handle.read()          // { events, ... }
const meta = handle.header ?? (await persistence.stat(sessionId))?.header
await handle.close()
```

本插件**早已**在 `lib/index.js:readStoredSession()` 里实现了这个双路径适配
（`inspect()` 优先，退回 `open()+read()`），所以迁移时直接复用，无需新代码。

---

## 三、`meta` 字段：宿主有显式白名单

这是本次移植最需要注意的一点。`agents.create` → `createAgent` → `sessions.prepare(id, {seed, meta, inheritedEventCount})`，
而 `SessionStore.prepare` 会**用一份显式白名单重建 header**：

```js
prepare(id, options) {
    let sessionId = /* 来自参数 id */;
    const meta = options?.meta;
    const header = {
        version: 4,                                                  // ← 自动补
        id: sessionId,                                               // ← 自动补
        createdAt: meta?.createdAt ?? Date.now(),                    // ← 自动补
        ...meta?.cwd === void 0 ? {} : { cwd: meta.cwd },
        ...meta?.parentSession === void 0 ? {} : { parentSession: meta.parentSession },
        isSeeded: meta?.isSeeded ?? false,
        ...meta?.origin === void 0 ? {} : { origin: meta.origin },
        ...meta?.delegationDepth === void 0 ? {} : { delegationDepth: meta.delegationDepth },
        ...meta?.agentPreset === void 0 ? {} : { agentPreset: meta.agentPreset },
    };
    return Session.create(sessionId, seed, header, options?.inheritedEventCount, this.projections);
}
```

要点：

- **白名单之外的键被静默丢弃**（不是报错）。
- `version` / `id` / `createdAt` **不需要也不能**自己传。
- `origin` 只接受 `"subagent"`；`delegationDepth` 必须是非负安全整数。
  两者都只在源 header 确实带该字段时才透传。

### `isSeeded` / `inheritedEventCount`：刻意都不传

`Session` 构造器里有这几条约束：

```js
if (this.header.isSeeded && suppliedInheritedEventCount === void 0)
    throw new Error("seeded session requires an inherited event count");
if (!this.header.isSeeded && inheritedEventCount !== 0)
    throw new Error("unseeded session inherited event count must be 0");
if (mode === "snapshot" && this.header.isSeeded
    && inheritedEventCount !== this.log.length && !markedSeed)
    throw new Error("seeded session constructor seed must equal its inherited prefix or mark its inherited cut");
```

官方 fork 走的是「截取前缀 + 插标记」路线，因此它传 `isSeeded: true` 且用
`buildForkSeed` 在 `inheritedEventCount` 位置插入 `session/end-seed { inherited: true }`：

```js
export function buildForkSeed(events, boundary) {
    const prefix = events.slice(0, boundary + 1);
    prefix.push({ type: "session/end-seed", seq: boundary + 1, time: events[boundary].time, data: { inherited: true } });
    return prefix.concat(openTurnClosers(prefix, { kind: "forked" }));
}
```

本插件是**全量日志复制**（没有裁剪前缀），因此最简且合法的路径是
**完全不传 `isSeeded`**：`prepare` 补 `false`，`inheritedEventCount` 缺省为 `0`，
校验退化成 `0 !== 0` 不成立 → 自然通过。全量 seed 仍会被构造器逐条校验并 `deepFreeze`。

> ⚠ 若将来改为「部分前缀」迁移，必须改走 `buildForkSeed` 并显式传
> `isSeeded: true` + `inheritedEventCount`。

---

## 四、一处自我更正（重要）

### 我最初的错误结论

读 `dsh-session` 的 `validateSessionHeader` 时，我看到：

```js
if (Object.hasOwn(record, "seedLength"))
    throw new Error('session header has invalid field "seedLength"');
```

于是断言：**上游把 `seedLength` 写进 `meta` 会导致迁移 100% 失败**。

### 实测推翻

我构造了两个用例对照（用真实 374 事件会话，逐字复刻 `prepare` 的 header 合成）：

```
直接把含 seedLength 的 header 交给 Session.create    →  失败：session header has invalid field "seedLength"
但真实路径 SessionStore.prepare 会静默丢弃 seedLength  →  通过
```

`prepare` 的白名单**先过滤掉**了 `seedLength`，`validateSessionHeader` 根本看不到它。
所以上游的写法在 rc.2 上**不会失败**——只是那条「血缘长度」信息实际丢失了
（rc.2 的血缘由 `parentSession` 表达）。

### 处置

- 本插件仍**不写** `seedLength`：既然宿主会丢弃，写上只会误导后来阅读者。
- 在 `lib/index.js` 的文件头注释里保留了完整更正记录，措辞明确标注「最初判断错误」。

**教训**：只读断言函数、不走完整调用链，会得出错误结论。这里的调用链是
`agents.create → createAgent → sessions.prepare → Session.create`，中间每一层都可能
改变或不改变传入值。**验证必须沿真实路径走**。

---

## 五、实测证据

### 5.1 真实会话日志校验

用磁盘上真实会话（`~/.dsh/sessions/**/session.v4.jsonl.zstd`，374 / 450 事件两个样本），
按 `prepare` 的合成逻辑构造 header 后交给真实 `Session.create`：

```
★ 本插件 meta（cwd + parentSession）        通过 ✓
      header = {"version":4,"id":"session-q-t-…","createdAt":…,
                "cwd":"C:\\x","parentSession":"p","isSeeded":false}
      log.length = 374   inheritedEventCount = 0
本插件 + agentPreset                        通过 ✓
本插件 + origin:subagent                    通过 ✓
```

### 5.2 宿主端端到端（mock ctx，真实 450 事件会话）

```
mode=keep:    agents.create → attachSession
              seed 450/450 事件逐条一致 ✓
              meta 无 seedLength ✓

mode=archive: agents.create → attachSession → archiveSession ✓

错误分支:     同区 → same-workspace ✓
              目标不存在 → target-not-found ✓

migratedTitle: 「我的会话」+ 同名存在        → 我的会话 [MS1]
               「我的会话」+ 已有 [MS1]      → 我的会话 [MS2]
               「我的会话 [MS1]」+ 已有 [MS1] → 我的会话 [MS2]   ← 替换而非累积
```

### 5.3 真实 DSH loader

```
loadProfileDirectory('dsh', <desktop profile>, <asar anchor>)
  layers        : 13
  skippedBundles: 0
  13 个 bundle 兼容性检查全部 OK
```

---

## 六、与上游的行为差异（本插件的取舍）

| 项 | 上游 | 本插件 | 原因 |
| --- | --- | --- | --- |
| HTTP 路由 | 新开 `POST /api/dsh-move-session/move` | 复用 `/archived/api` 的 `move` 方法 | 本插件已有同源 + loopback 校验（`isTrustedApiRequest`），不必再建一套 |
| 日志读取 | `persistence.readFrom(id, 0)` | `readStoredSession()` | `readFrom` 在 rc.2 不存在 |
| `meta.seedLength` | 写入 | 不写 | 宿主白名单会丢弃，写了误导 |
| `meta.version` | 未写 | 未写 | 由 `prepare` 自动补；传了也无意义 |
| 侧边栏行菜单注入 | 用 ARIA 锚点注入「迁移会话」项 | 未做 | 会话头部入口已覆盖主要场景；行菜单注入依赖 React fiber 反解，脆弱且本次未验证 |

---

## 七、未验证 / 已知边界

- **未做真实迁移的落盘验证**。上述都是「真实数据 + 真实校验函数 + mock 宿主服务」，
  没有在运行的 DSH 上真的迁移一次并检查磁盘上的新会话目录。
  首次实际使用时建议迁移一个**不重要的短会话**观察。
- **附件与工作区文件不迁移**：只复制会话日志，日志内的附件引用原样保留
  （与上游一致）。
- **副本以 live agent 形式驻留内存**：与官方 fork 行为一致，不随插件卸载。
- **运行中会话拒绝迁移**：客户端置灰 + 宿主 `agent.status === "running"` 二次校验，
  返回 HTTP 409（`session-busy`）。
- **目标目录预检**：迁移前 `stat` 目标工作区路径，失效目录（如临时目录被清理）
  在任何写入前拒绝，避免产生孤儿会话（`target-missing-dir`）。
