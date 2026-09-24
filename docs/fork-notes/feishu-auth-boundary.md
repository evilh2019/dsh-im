# Fork 边界说明：飞书外部授权集成 vs 可上游贡献

> 目的：这条 fork 长期维护两件容易混在一起的事。混在一起会导致被上游拒绝（见 #117 / #67）。

## 两条线

| 线 | 分支 | 内容 | 能否上游 |
|---|---|---|---|
| **A. 可上游** | `feishu-card-order-only` | 飞书卡片顺序 / 延迟发送 / 换卡；只依赖项目内 `approval/requested` + `interaction.respond` | ✅ 可以 |
| **B. 仅 fork** | `upgrade-4.20.0-v2` | A 的全部 + **外部授权集成**（`auth_request.py` / `FEISHU_AUTH_DIR` / `authDir/<auth_id>.json`） | ❌ 不可 |

## 为什么 B 不能上游

上游维护者在 #117 已明确：

> 该 PR 依赖仓库外部的 auth_request.py，并在核心 Bridge 中引入了 FEISHU_AUTH_DIR、authDir/<auth_id>.json
> 以及文件轮询式的审批协议。由于该外部系统不属于本项目……**本项目不能在核心代码中直接集成或维护这套外部约定**。
> 飞书审批卡片如果继续推进，希望只基于项目内的正式接口实现，并移除 auth_request.py、FEISHU_AUTH_DIR、
> authDir 文件状态回调及相关假设。**如果确实需要兼容该外部系统，更合适的方式是将它维护为独立 adapter/plugin，
> 而不是合入核心 Bridge。**

#67 另有 5 条阻塞项（`formValue` 解析回归 / 分支与 `lib` 冲突 / 调试日志洪泛 / 配置链不完整 / `authId` 校验与并发）。

## 为什么"独立 adapter/plugin"目前做不到

`dsh-im` 自身是一个 DSH bundle（`package.json` 的 `dsh.bundle`），`exports` 仅暴露 `.` 与 `./client`，
**没有面向第三方的 channel / card-action 扩展点**。因此外部授权集成在今天的 dsh-im 上只能以
"fork 补丁"的形式存在——它已经不在上游核心里，符合维护者要求的边界。

要真正做成 adapter，需要上游先提供扩展点（如 card-action hook / adapter 接口）。

## 纪律

1. **不要从 fork 线（B）直接开 PR。** 任何上游提交都必须从 A 线切出，并确认 `grep -rn "authDir\|auth_request\|FEISHU_AUTH_DIR" src plugin-src` 无命中。
2. 提交上游前先跑 `gh pr list --state all` 看既有 PR 与维护者意见（本仓库已有 #67 / #117 两条同源线程）。
3. 验证口径：完整 `npm ci` 依赖树下的全量套件，与 `origin/main` **逐项相同**（tests/pass/fail/cancelled 四个数）。
   曾在依赖残缺的 checkout 上验过，那次"零回归"读数不可信。
