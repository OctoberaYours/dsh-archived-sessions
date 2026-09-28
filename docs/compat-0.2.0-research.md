# archived-sessions 在 DSH 0.2.0-rc.1 上的两条隐患 —— 修法研究

> 研究时间：2026-09-29。基于 asar 内代码逐行核对（已解包到 `%TEMP%\dsh-asar`）。
> 结论分两类：**可修** 与 **上游固有、本插件不应越界**。

---

## 隐患 1：删除「已加载但未运行」的会话后，agent 残留在 `agents` 注册表

### 现状

插件 `lib/index.js` 的删除流程（约 1066-1073 行）只做了：

```js
ctx.get("sessions").liveEntryFor(sessionId)  →  sessions.detachEntered(entry)
```

即**只摘 sessions 注册表**。`agents` 注册表没动。

子代理报告的后果：`agents.get(id)` / `agents.list()` 在删除后仍返回该 agent（内存残留，重启才清）。

### 根因（已定位到行）

`dsh-agent-loop/lib/index.js` 里 agent 与 session 是**配对注册**：

```js
// 1748-1749  publish()
detachSession = agent.ctx.sessions.enter(session);
detachAgent  = loopCtx.agents.enter(agent, parentAgent);
```

**配对注销**就在同一个 `dispose` 闭包里：

```js
// 1700-1702
try {
    detachAgent?.();     // 摘 agents
    detachSession?.();   // 摘 sessions
} finally { ... }
```

`dispose` 是个**闭包内部函数**，只有两个触发点：

| 触发点 | 行号 | 说明 |
|---|---|---|
| `machine.scope.rawDispose` | 1716 | agent 自身 cordis scope 结束 |
| owner fiber 卸载 | 1705 `await unfollowOwner()` | 经 `ownerCtx.effect(...)`（1713-1722） |

而 `ownerCtx.effect` 的清理函数（1717-1721）在 owner 被 dispose 时调用 `dispose(true)`。

### 关键判断：插件**不应**、也**无法**主动 dispose

1. **无法**：`dispose` 不在任何服务对象上，插件拿不到。`AgentLoop` 类（`dsh-agent-loop:1523`）的公开方法只有
   `constructor / reportConfiguredStartupFailure / restoreOrCreateConfigured /
   waitForDrainingConfiguredIdentity / prepare / create / createStoredSession /
   appendUnstoredSuffix / createAgent / setupAndPublish / initializeAgent / resume / resumeWith` ——
   **没有**任何按 id 处置 agent 的方法。子代理所说的 `AgentLoop.disposeAgent` 确实不存在。

2. **不应**：官方自己的做法就是**只 cancel 不 dispose**。`dsh-agent/lib/index.js:35`：

   ```js
   if (agent?.status === "running") agent.cancel({ kind: "user" });
   ```

   即：已停止（idle）的 agent 留在注册表里，是**上游的既定行为**，不是本插件引入的缺口。
   agent 的销毁钩子挂在 owner fiber 的 effect 上，语义上属于「owner 生命周期」而非「删除会话」。

3. **可以做到的部分**：`AgentRegistry.store` 是**公开字段**（`dsh-agent/lib/index.js:323` `store = new Map();`，
   不是 `#store`），所以插件其实**能**拿到 entry：

   ```js
   const agents = ctx.get("agents");
   const entry = agents?.store?.get(sessionId);
   if (entry) agents.detachEntered(entry);   // dsh-agent/lib/index.js:538
   ```

   这会让 `agents.get(id)` 返回 undefined，并发出 `agent/disposed` 事件。

### 建议

**倾向「不修」**，理由：

- 上游语义如此（idle agent 常驻），插件强摘可能与 owner fiber 的后续清理冲突
- `store` 是**实现细节**（虽公开但未文档化），直接 `detachEntered` 属于跨层操作
- 危害有限：**内存残留**，不产生错误行为；且只影响「打开过、已停止、又被删除」这条窄路径
  （真正归档、本进程未加载的会话不受影响，因为 `agents.get` 本来就返回 undefined）

**若确实要修**，保守写法是「只摘注册表、不碰 agent 本身」，并在注释里写明这是补偿上游不配对的行为：

```js
// 补偿：上游 agent/session 是配对注册（dsh-agent-loop:1748-1749），
// 但 dispose 只能由 owner fiber 触发，删除会话时不会自动摘 agents。
// 这里只摘注册表让 get()/list() 不再返回它；agent 自身的 scope 交给 owner。
// store 是公开字段但非文档化契约，故做能力探测。
const agents = ctx.get("agents");
const entry = agents?.store?.get?.(sessionId);
if (entry !== void 0 && typeof agents.detachEntered === "function") {
    agents.detachEntered(entry);
}
```

**并且必须实测**：删一个有 agent 的会话，确认 (a) 列表里消失 (b) 无报错 (c) 后续还能重新打开该会话。

---

## 隐患 2：删盘依赖插件自己重算的上游私有布局

### 现状

`persistence.remove` 在 0.2.0-rc.1 **不存在**（`dsh-session-persistence` 与
`dsh-session-persistence-jsonl` 两个包里定义数均为 0）。插件的应对：

```js
// lib/index.js:798
const location = persistence !== void 0 && typeof persistence.locate === "function"
    ? persistence.locate(meta) : void 0;
// 不可用时退回 L803-806 sessionDirFor(meta)
```

`sessionDirFor`（L226-230）用插件自己的 `projectKey` / `encodeSegment` 复刻官方 jsonl 布局，
再经 `insideSessionsRoot` 围栏（L793-797）与扩展名正则（L829-833）删除日志。

### 评估

- **这次升级无影响**：子代理实测布局比对 **0 处不符**，且两个 persistence 包在
  `0.1.7-rc.2` → `0.2.0-rc.1` 之间**字节完全相同**（升级没动布局）。
- **结构性问题**：这是插件对上游私有布局的**第二份实现**。上游改布局 → 删除静默失效或删错目录。
- **已有的缓解**：定位不到时插件**显式报 404**（L810-815）而不是假装成功 —— 避免了「报告删了但还在」。

### 建议

**不修**。没有官方删除原语可用，重算是唯一选择；且上游改布局是小概率事件，
真发生了插件会以 404 暴露（而非静默错删）。可以做的是**加一条注释**指向这个风险。

---

## 附：本次核对过的、确认无问题的点

| 项 | 结论 |
|---|---|
| 服务存在性 | `sessions` / `sessionPersistence` / `workspaceRegistry` / `agents` / `webServer` / `slots` / `locale` 全部存在 |
| `sessionPersistence` 实现位置 | 在 `dsh-session-persistence-jsonl/lib/index.js`（142847 B），基类包仅 12920 B |
| 已移除 API 的守卫 | `readRaw` / `remove` / `artifactInfo` / `coordinator` 四处均有 `typeof` 或可选链守卫 |
| `workspaceRegistry` | `archiveSession` / `requireState` / `setState` 存在；`deleteSession` 不存在（插件已改用别的方式） |
| peer 范围 | 11 个 peer 全改为 `^0.1.7-rc.2 \|\| ^0.2.0-rc.1`，门禁实测通过 |

## 附：本次已修

`lib/client.js` 的 `current` 恒 undefined 问题（`98e0a6c`）：
sessions store 自 0.1.7-rc.2 起无 `current` 字段，改用官方判据 `retainedBy.mainView`。
