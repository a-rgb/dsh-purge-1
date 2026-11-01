# FORK NOTES — dsh-purge 1.1.71 维护副本（a-rgb）

上游：<https://github.com/YuJunZhiXue/dsh-purge>（MIT，见 `LICENSE`）

本仓库版本：**dsh-purge 1.1.71**，基于 `mmparko/dsh-purge` 的 `v1.1.71` tag
（上游已删除 v1.1.71 / v1.1.72 的 tag 与 release，官方最老 tag 现为 v1.1.73）。

两条线各自独立，`master` 是上游 1.1.86 原样，本分支是从 1.1.71 长出来的自用版。

## 为什么停在 1.1.71 而不是跟到 1.1.86

1. `skills/redteam/` 的 23 个技能文件在 1.1.73 之后被整目录删除，1.1.86 一个都不剩。
   对演练台而言这是功能性倒退。
2. 1.1.71 的补丁面最小（82 条 vs 1.1.86 的 85 条），需要打补丁的宿主点位最少。
3. 宿主是 0.2.1-rc.5；1.1.80 的发布说明明确写「0.2.1-rc.3 及更新的 rc 不支持」，
   之后 1.1.83/1.1.84 才又补上 rc.4/rc.5 的适配。1.1.71 只需要补三处即可工作。

## 一、修复 1.1.71 在宿主 0.2.1-rc.5 上的三个真实缺口

### 修复 A — `#40 COMPLETE_PROMPT_KEEP_INJECT` 命中不到 0.2.1 的 complete 收尾块

0.2.1 的 `SystemPrompt.assemble()` 在收尾块前多插了一行 `const requiredContexts = new Set(...)`，
`contexts` 分支也从清空（`[]`）变成 `filter(requiredContexts.has)`。原有 7 个 pattern 全部
`not_found` → `patchStatus` 返回 **`skipped`（不是 `pending`）**，于是：

```
ensureHostClean() → needsApply 被跳过 → 标记永不写 → hostPromptKeepsInject() 恒 false
→ allPatchHostsWritten() 恒 false → 重启门禁永久输出「清洗没有写进全部宿主，已取消重启」
```

修法（`lib/core.js`）：新增 `COMPLETE_PROMPT_WATERFALL_INJECT_02` 精确匹配 0.2.1 的
`requiredContexts` 形状；再补一条**兜底 pattern**，未来任何形状只要还留着
`sections: completeSection === void 0 ? transformed.sections : [completeSection],`
就就地改成保留 dsh-purge inject 段。两条都幂等。

### 修复 B — `hostFs` 缺 fd 类与路径类同步方法

`lib/core.js` 里 `const fs = hostFs`，而 `HOST_SYNC` 缺 `closeSync` / `readSync`：

- `backupAll()` 的 `fs.closeSync(placeholderFd)` 抛 `TypeError`（被 `try/catch` 吞）→ 占位 fd 泄漏
  → 目标文件被本进程占着 → `rename(tmp, *.dshpurge.bak)` 每个文件 `EPERM`
  → 全量备份 **0 条**、`hasBackup()` 恒 false、`revert` 没有原件可回；
- `isTextShim()` 的 `fs.readSync` 每次抛 → 一律判「非文本 shim」→ `dsh.ps1` / `dsh` 永不注入。

修法（`lib/extract-asar.js`）按上游 `#150` 的形态分两张表：

```js
const HOST_SYNC = [...路径语义..., "accessSync", "truncateSync", "utimesSync"];
const HOST_FD   = ["closeSync", "readSync", "writeSync", "fsyncSync", "fstatSync", "futimesSync"];

for (const m of HOST_SYNC) hostFs[m] = route(m, false);          // 走 asar 探针
for (const m of HOST_FD)   hostFs[m] = (fd, ...a) => nodeFs[m](fd, ...a);  // fd 语义，不能过 route
```

**关键**：fd 方法不能塞进 `HOST_SYNC`。`route()` 会先拿第一个参数当路径跑 `asarShadowsPath()`，
对数字 fd 做无意义的 asar 探测，还可能冲掉探针缓存。

### 修复 C — `cordis.patch.yml` 不再重复 insert 本体

1.1.75 之前会把本体 `dsh-purge` 在 `- insert:` 下再插一遍。用户层
`profile/cordis.patch.yml` 已有同名条目，两份重复 → 宿主报 `already queued: dsh-purge`。

修法：删掉 `- insert:` 下的本体条目，把 `inject` 与全部 `config` 挪到 `- id: dsh-purge`
覆盖条目上（与上游 1.1.75+ 同形态）。演练台三条子行保留在原 `- insert:` 下。

## 二、从 1.1.86 反向移植的功能

原则：只取**自包含**模块，不碰 1.1.71 与 1.1.86 差异过大的核心（`core.js` / `index.js` 主体）。

| 模块 | 来源 | 依赖 | 状态 |
|---|---|---|---|
| `lib/anchor.js` | 1.1.86 | 无本地 import（自带子会话判定） | ✅ 逐字移植 |
| `lib/subagent-control.js` | 1.1.86 | 仅 `./anchor.js` | ✅ 逐字移植 |
| `lib/apply-progress.js` | 1.1.86 | 无 | ✅ 逐字移植 |
| `lib/mnemon-settings.js` | 1.1.86 | 无 | ✅ 逐字移植 |
| `lib/profile-insert.js` | 1.1.86 | 无 | ✅ 逐字移植 |
| `lib/preset-default.js` | 1.1.86 | 仅 `core.findDshHome`（1.1.71 有） | ✅ 逐字移植 |

### 新增能力 1 — 尾部锚点（recency 位）

注入正文约 22k 字符落在 system prompt **最前**，宿主后面还跟着工具描述和运行上下文，
模型最后读到的不是我们。锚点把身份与格式规则补到 sections 数组**末尾**。

接法（两处）：

- `lib/identity.js`：`normalizeRewriteOptions` 两个返回分支各加 `anchorText`；
  `rewritePromptAssembly` 返回前把 `ANCHOR_SECTION` 物理追加到 `outSections` 末尾
  （waterfall 按数组顺序渲染，`order` 不参与排序，所以只能物理追加）。
- `lib/index.js`：`rewritePromptAssembly` 调用处传 `anchorText: enabled ? anchorTextFor(config) : ""`；
  `applyHostHooks` 里 `registerAnchor(ctx, cfg)` + `ctx.effect` 回收。

锚点规则跟随操作者语言：中文请求回中文，锚点本身是英文**不算**语言指令。
这不是装饰——1.1.84 之前红队模式回答变英文的直接原因就是写死了英文开头句。

回滚开关：`anchorMode: "off"`。

### 新增能力 2 — 子代理审批接管

宿主给子会话写死 `approval/policy = "never"`，这一步发生在 `approval/request` 瀑布**之前**，
任何监听器都救不到：

```
dsh-subagent captureDelegatedPolicyOverrides → approvalPolicy: "never"
dsh-user-approval decide() → if (effectivePolicy === "never") return "rejected";
```

顺序只能是：先把 policy 翻成 `ask`（`session/event` 观察者 + `queueMicrotask` 补 append），
waterfall 才会跑；再在 `approval/request` 里对子会话放行一次。只作用于子会话，主会话零影响。

回滚开关：`subagentControl: false`。

## 三、验证方式

`core.ALL_PATCHES` 与 `applyReplacementsToText` 都是导出项，可以直接拿 `app.asar` 里
**真实的** `assemble()` 代码块喂进去逐条试，比读源码猜可靠得多。

```js
const src = fs.readFileSync(".../resources/app.asar", "utf8");
const idx = src.indexOf("const requiredContexts = new Set(");
const start = src.lastIndexOf("const transformed = await this.ctx.waterfall", idx);
const real = src.slice(start, src.indexOf("};", idx) + 2);
const p40 = core.ALL_PATCHES.find((p) => p.id === 40);
const r = core.applyReplacementsToText(real, p40, "system-prompt");
```

实测结果：

```
[#40] changed = true / 保留 requiredContexts = true / 保留 contexts filter 语义 = true
[#40] 裸 [completeSection] 已消失 = true / 幂等 = true
[hostFs] closeSync / readSync / writeSync / fsyncSync / fstatSync / accessSync = function
[anchor] sections 末位 = dsh-purge:anchor，含 OPERATOR ANCHOR，二次装配仍只 1 份
[anchor] anchorMode:"off" 与 enabled:false 均不出现锚点
[anchor] isSubagentContext 判定 origin=subagent / 空 header / null 均正确
```

**坑**：用 PowerShell 读 `app.asar` 数行号会与 Node 错位（换行符切分规则不同），
必须用 `indexOf` 按字符串定位。

## 四、没有移植的部分（明确说明）

| 1.1.86 的东西 | 为什么不要 |
|---|---|
| `lib/web-reclaim.js` / `bin-preflight.js` / `web-preflight-entry.js` | 只在 `dsh web` 形态有意义；本机是桌面端 |
| `lib/redteam/evidence-ledger.mjs` | 依赖 `store-core.js` 的大改（1.1.86 该文件重写 255/267 行），跟 1.1.71 的 store 不兼容 |
| `lib/default-prompt-inject.{zh,en}.md` | 1.1.71 用加密槽 `table-read.js` 解默认提示词，两者是两套机制，替换会破坏 `override-state` 判定 |
| `lib/core.js` 主体（+662/−99） | 差异面过大，混入无法逐条验证 |
| `test/` 12 个测试文件 | 断言的是 1.1.86 的行为，在 1.1.71 上跑必然失败 |
| 红队模式中文化、深色模式修色、`#194` 熔断 | 属 UI/文案层，与 `client.js` 强绑定，1.1.71 的 client bundle 不同 |

## 五、安装

```powershell
dsh plugin --profile desktop add https://codeload.github.com/a-rgb/dsh-purge-1/tar.gz/refs/heads/maintain/v1.1.71-yg
```

或本地 tarball：

```powershell
dsh plugin --profile desktop add file:<本地路径>/dsh-purge-1.1.71.tar.gz
```

**注意**：包内 `cordis.patch.yml` 的重复 insert 修复是本分支的改动。重装或升级该包会覆盖，
需要重新应用。

## 六、配置开关一览

| 键 | 默认 | 作用 |
|---|---|---|
| `enabled` | `true` | 总开关 |
| `anchorMode` | `"auto"` | `"off"` 关闭尾部锚点 |
| `subagentControl` | `true` | `false` 关闭子代理审批接管 |
| `stripMnemon` | `true` | 剥掉 mnemon 插件消息 |
| `capabilityProtocol` | `true` | 注入 skill / MCP 能力协议段 |
| `hooksDenyBypass` | `true` | hooks 拒绝放行 |

## 七、分支与 tag

```
maintain/v1.1.71-yg   → 当前维护线
v1.1.71-yg.1          → 首版（hostFs 塞错位置，勿用）
v1.1.71-yg.2          → hostFs 改用 HOST_FD
```
