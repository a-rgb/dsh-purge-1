# FORK NOTES — dsh-purge 1.1.71 维护副本（a-rgb）

上游：<https://github.com/YuJunZhiXue/dsh-purge>（MIT，见 `LICENSE`）

本仓库版本：**dsh-purge 1.1.71**，基于 `mmparko/dsh-purge` 的 `v1.1.71` tag
（上游已删除 v1.1.71 / v1.1.72 的 tag 与 release，官方最老 tag 现为 v1.1.73）。

维护动机：在 DSH **0.2.1-rc.5** 桌面端跑 1.1.71 时，有三个会让功能静默失效的问题。
本分支只做这三处修复，不改其它行为。

## 修复 1 — `#40 COMPLETE_PROMPT_KEEP_INJECT` 命中不到 0.2.1 的 complete 收尾块

0.2.1 的 `SystemPrompt.assemble()` 在收尾块前多插了一行 `const requiredContexts = new Set(...)`，
`contexts` 分支也从清空（`[]`）变成 `filter(requiredContexts.has)`。原有 7 个 pattern 全部
`not_found` → `patchStatus` 返回 **`skipped`（不是 `pending`）**，于是：

```
ensureHostClean() → needsApply 被跳过 → 标记永不写 → hostPromptKeepsInject() 恒 false
→ allPatchHostsWritten() 恒 false → 重启门禁永久输出「清洗没有写进全部宿主，已取消重启」
```

修法（`lib/core.js`）：

- 新增常量 `COMPLETE_PROMPT_WATERFALL_INJECT_02`，精确匹配 0.2.1 的 `requiredContexts` 形状，
  保留原语义（purge 段 + rest 尾 + complete 收尾，`requiredContexts` 过滤逻辑原样保留）。
- `#40` 追加**兜底 pattern**：任何未来形状只要还留着
  `sections: completeSection === void 0 ? transformed.sections : [completeSection],`
  就就地改成保留 dsh-purge inject 段。这类「官方改形 → skipped → 永久取消重启」不会再复现。

两条都幂等：已在 patched 文本上再跑一次 `applyReplacementsToText` 不再变化。

## 修复 2 — `hostFs` 缺 `closeSync` / `readSync`（静默失效，不抛错）

`lib/core.js` 里 `const fs = hostFs`，而 `lib/extract-asar.js` 的 `HOST_SYNC` 只有 `openSync`：

- `backupAll()` 的 `fs.closeSync(placeholderFd)` 抛 `TypeError`（被 `try/catch` 吞）→ 占位 fd 泄漏
  → 目标文件一直被本进程占着 → `rename(tmp, *.dshpurge.bak)` 每个文件 `EPERM`
  → 全量备份 **0 条**、`hasBackup()` 恒 false、`revert` 没有原件可回；
- `isTextShim()` 的 `fs.readSync` 每次抛错 → 一律判「非文本 shim」→ `dsh.ps1` / `dsh` 永不注入。

修法：`HOST_SYNC` 补 `closeSync / readSync / writeSync / fsyncSync / fstatSync / accessSync`。

## 修复 3 — `cordis.patch.yml` 不再重复 insert 本体

1.1.75 之前版本的包内 patch 会把本体 `dsh-purge` 在 `- insert:` 下再插一遍。用户层
`profile/cordis.patch.yml` 已经有同名条目，两份重复会让宿主报 `already queued: dsh-purge`。

修法：删掉 `- insert:` 下的本体条目，把 `inject` 与全部 `config` 挪到 `- id: dsh-purge`
覆盖条目上（与上游 1.1.75+ 同形态）。演练台三条子行保留在原 `- insert:` 下。
`personaSuffix` 维持 mmparko 版的 `Your working directory is {{cwd}}.`。

## 验证

对真实宿主代码（`resources/app.asar` 里的 `assemble()` 块）实测：

```
[#40] changed = true
[#40] 保留 requiredContexts = true
[#40] 保留 contexts filter 语义 = true
[#40] 裸 [completeSection] 已消失 = true
[#40] 含 dsh-purge 保留逻辑 = true
[#40] 幂等（二次应用不再变化）= true
[hostFs] closeSync/readSync/writeSync/fsyncSync/fstatSync/accessSync = function
```

`cordis.patch.yml` 解析为合法顶层数组，`insert` 仅含演练台三条，本体重复 insert 为 0。

## 与上游的差异面

```
lib/core.js           # 修复 1（+39 行）
lib/extract-asar.js   # 修复 2（+6 行）
cordis.patch.yml      # 修复 3（去重）
FORK-NOTES.md         # 新增
```

## 安装

```powershell
dsh plugin --profile desktop add https://codeload.github.com/a-rgb/dsh-purge-1/tar.gz/refs/heads/maintain/v1.1.71
```

或本地 tarball：

```powershell
dsh plugin --profile desktop add file:<本地路径>/dsh-purge-1.1.71.tar.gz
```

## 回滚

```powershell
gh api -X DELETE repos/a-rgb/dsh-purge-1/git/refs/heads/maintain/v1.1.71
# 或本地 git：git checkout master && git branch -D maintain/v1.1.71
```
