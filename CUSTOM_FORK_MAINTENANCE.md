# Handy 自定义 Fork 维护手册

本文档记录 `luweiCN/Handy` 自定义分支的产品约束、实现差异，以及以后将 `cjpais/Handy` 最新主分支合并进来时应执行的流程。目标读者是未来接手仓库的 AI 或开发者。

> 先完整阅读仓库根目录的 `AGENTS.md` 和本文档，再修改、合并、构建或安装。代码和测试是运行事实；如果它们与本文档不一致，应先查明原因，而不是静默选择其中一个，并在验证后更新本文档。

## 1. 当前维护范围

截至 2026-08-25：

| 项目               | 当前值                                                                              |
| ------------------ | ----------------------------------------------------------------------------------- |
| 用户 fork          | `https://github.com/luweiCN/Handy`                                                  |
| `origin`           | `https://github.com/luweiCN/Handy.git`                                              |
| `upstream`         | `https://github.com/cjpais/Handy.git`                                               |
| 自定义分支         | `feature/long-dictation-stable-postprocess`                                         |
| 最后合入的上游基线 | `20ada47`，本地引用为 `upstream/main`                                               |
| 自定义功能基线     | `3292f1e`；独立身份修改位于其后的当前分支 HEAD                                      |
| fork macOS bundle  | `Handy Fork.app` / `com.luweicn.handyfork` / `0.9.7-fork.3` / Apple Silicon `arm64` |
| 官方 macOS bundle  | `Handy.app` / `com.pais.handy`；与 fork 并列安装                                    |
| fork 更新渠道      | `luweiCN/Handy` releases；自有 Tauri minisign key，与官方 channel/key 分离          |
| 本地签名           | ad-hoc；没有 notarization，也没有 Apple Developer Team ID                           |
| 当前 LLM 配置      | DashScope OpenAI-compatible endpoint + `qwen3.7-flash`                              |

上表是快照，不是永久常量。每次合并、构建和安装后都应更新日期、上游基线、版本及 artifact hash。

## 2. 不可破坏的产品行为

后续合并可以改实现，但除非用户再次明确改变决定，必须保留以下行为。

### 2.1 Qwen3-ASR 长录音必须可用

- 故障基线是 `transcribe-cpp transcription failed: ... decode hit the context/generation cap ... (status 18)`。
- 当时的 Qwen3-ASR decoder 每次生成固定最多 256 tokens。模型拥有更大的 context，并不代表 Handy 当前 backend 自动拥有更大的输出预算。
- 当前方案只对 `qwen3_asr` 生效：音频超过 30 秒后，在每个 25–30 秒区间寻找最低能量的 100ms 窗口，并在静音附近切段；各段依次转录后拼接。
- 该方案不修改 `transcribe.cpp` ABI，不维护第二个底层 fork，并能通过增加分段支持总时长远大于单段上限的录音。
- 任一分段失败时整次操作返回错误，不把不完整的部分文本冒充完整结果。
- 主要实现和测试位于 `src-tauri/src/managers/transcription.rs`。

如果未来上游暴露了 `max_new_tokens`，不能仅因为“可以调大”就删除切段。先比较长时间录音、内存占用、速度、分段边界质量和失败恢复；固定 4096 仍是有限上限，且 decoder KV cache 预算明显高于 256。

### 2.2 不提供软件伪造的 Qwen streaming

- Qwen3-ASR 0.6B/1.7B 权重本身可以用于 streaming，但本 fork 使用的 GGUF + `transcribe-cpp` backend 没有 Qwen stream hooks。
- 曾实现 Handy rolling preview，提交为 `42442e7`；用户认为这不属于模型/backend 的原生能力，因此已由 `74b319c` 完整撤销。
- 两款 Qwen 的 catalog capability 必须保持 `streaming: false`，除非未来所用 inference backend 真正实现 `stream_begin`、`stream_feed`、`stream_finalize` 一类原生状态接口。
- 不要通过定时重复转录最近几秒音频来冒充 native streaming。

### 2.3 LLM 后处理以速度优先，但不自动截止

当前所有远程 provider/model 共用可配置的 hedged-request 策略。这里的 request 指一个 logical attempt；单个 attempt 内部还可能发生 reasoning-field 兼容重试。默认值必须保持原有速度优先行为，但用户可在 Post Processing → Request Acceleration 中独立关闭两层加速：

1. “Parallel Requests”默认开启：`t=0` 向当前选中的同一 endpoint/model 同时发送两个相同请求；关闭时只发送一个。
2. “Delayed Backup Request”默认开启：如果设定时间内没有得到非空成功结果，再发送一个相同请求；默认 5 秒，可调范围 1–30 秒。即使已发请求很早报错或返回空内容，备用请求也不能提前到设定时间之前。
3. 第一个非空成功结果获胜；错误或空响应不能抢占成功结果。
4. 获胜后在客户端 abort 其余任务。
5. chat completion 没有 response/generation deadline；供应商很慢时继续等待，由用户手动取消。

四种组合的 logical attempt 上限分别是：两项都关 = 1；只开 Parallel = 2；只开 Delayed = 1 + 1 delayed；两项都开 = 2 + 1 delayed。开关只改变 fan-out，不得改变 winner、错误、空响应、取消或无 chat deadline 的语义。

边界条件：

- 连接建立仍有 3 秒 `connect_timeout`，它不限制连接成功后的生成时间。
- `/models` metadata 请求单独保留 8 秒 timeout，因为该操作没有同等清晰的长期等待/取消体验。
- 客户端取消或 abort 不保证供应商停止服务端推理，也不保证停止计费。按用户配置，一次后处理应按最多 1–3 个可能完整生成/计费的 logical attempts 估算费用和配额。
- 未识别 endpoint 首次拒绝 `reasoning_effort` 时，每个并行 attempt 都可能先收到 400/422 再发一次无 reasoning 字段的兼容请求，因此物理 HTTP 请求数可能超过三个；被拒绝的探测通常未生成内容，但仍会消耗请求配额。
- 三次请求只发给用户当前选择的同一 endpoint/model，不自动把私人文本扩散到第二家供应商。
- 正常录音和历史重新后处理都必须使用现有 cancellation generation，确保用户取消后 drop 正在等待的 future。

2026-08-22 的用户实际体验是：安装当前组合修改后，尚未再遇到后处理偶发持续几十秒的情况。这只能证明组合版本的体感改善，不能单独证明 shared client、connect timeout、reasoning 参数或 hedging 中哪一项的因果。从机制和延迟量级判断，几十秒尾延迟消失更可能主要来自 hedging；shared client 通常只节省 DNS/TCP/TLS 建连和连接池开销，3 秒 connect timeout 则主要约束建连故障。

Hedging 是用户为“速度优先”明确选择的 fork-only 政策。它会必然增加上游请求、费用和服务端负载，因此不应默认放入 Handy 官方 PR，也不应与连接复用、provider 参数或历史 UI 混成一个提交。

2026-08-23 用户进一步报告：修改前每天多次后处理卡住 10–20 秒，当前 hedging 版本在一天正常使用中零次复现。已在官方 Ideas 发布 [Discussion #1953](https://github.com/cjpais/Handy/discussions/1953)，征求默认关闭 opt-in 方案的社区反馈。维护者 `cjpais` 随后[明确回复不会合入](https://github.com/cjpais/Handy/discussions/1953#discussioncomment-18123574)：该方案过于特定，会增加费用、UI 和理解复杂度，而且项目未来倾向本地模型，不希望继续扩张远程模型功能。因此不创建对应的上游代码 PR；自用 fork 保留激进默认值，但从 2026-08-25 起允许用户关闭任一加速层并调整 delayed 秒数。

供应商关闭 thinking/reasoning 的字段并不统一。当前实现位于 `src-tauri/src/llm_client.rs`：

- DashScope：`enable_thinking: false`
- DeepSeek 官方 endpoint：`thinking: { "type": "disabled" }`
- OpenRouter：`reasoning: { "effort": "none", "exclude": true }`
- 未识别的 OpenAI-compatible endpoint：先尝试 `reasoning_effort: "none"`；若 400/422 拒绝，再不带该字段重试，并在当前进程内记住拒绝结果。

不要假设火山方舟或未来供应商接受上述任意一个字段；新增 endpoint 时应依据其官方 API 和本地 mock 测试增加明确适配。

### 2.4 历史记录必须能从原始转录重新后处理

- 历史页的 Sparkles 按钮调用 `retry_history_entry_post_process`。
- 每次都以保存的 `transcription_text` 原始 ASR 文本为输入，并使用当前 provider、model 和 prompt。
- 不重新读取或转录音频，也不把旧的 `post_processed_text` 再次送入模型，避免反复润色累积偏差。
- 成功时只替换 `post_processed_text`、`post_process_prompt`，并设置 `post_process_requested = 1`；原始 `transcription_text` 必须保持不变。
- 失败或取消时不得清空、覆盖现有处理结果。
- 处理期间按钮变为取消操作；图标按钮必须保留 accessible name。一次性操作不能错误标记为 `aria-pressed` toggle。
- 页面优先展示后处理结果，同时通过原生 disclosure 保留“原始转录”查看入口。
- 新增或改名文案时必须同步全部 locale，并通过翻译一致性检查。

## 3. 自定义提交账本

这些提交按顺序位于自定义分支。判断当前行为时要看整条提交链，不能只看早期 commit 标题。

| Commit    | 状态           | 含义                                                                                                                                     |
| --------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `5cb34ef` | 当前有效       | `fix: chunk long Qwen dictations`；Qwen 长音频静音感知切段                                                                               |
| `da60182` | 部分有效       | `fix: bound post-processing latency`；共享 HTTP client、连接超时、reasoning 字段和错误诊断仍有效，早期 response timeout 已被后续提交移除 |
| `42442e7` | 已撤销         | `feat: preview Qwen dictation live`；软件 rolling preview，不应恢复                                                                      |
| `74b319c` | 当前有效       | 精确 revert `42442e7`，恢复 Qwen `streaming: false`                                                                                      |
| `f8593a5` | 被后续策略调整 | 引入 `t=0` 双发和 delayed third request；早期 3 秒/8 秒时序已被后续提交替换                                                              |
| `1bfadc3` | 当前有效       | 第三发改为 5 秒；移除 chat response deadline；保留手动取消、3 秒 connect timeout 和 8 秒 metadata timeout                                |
| `1881d5a` | 当前有效       | 历史记录从原始转录重新执行 LLM 后处理，含 UI、取消、数据更新、bindings 和 24 个 locale                                                   |

### 3.1 上游贡献拆分策略

上游当前处于 feature freeze，并明确要求一个 PR 只包含一个 fix 或 feature。本 fork 的长音频、LLM 和历史功能不得打包成一个官方 PR，应按以下边界处理：

| 功能               | 上游策略                                                                                                                                                                                                         |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Qwen 长音频切段    | 不重开已关闭的 Handy PR #1882。优先用真实 status 18 回归证据支持 `transcribe.cpp` #74 与 Handy #1633；待两个分支重新无冲突后，再用私密长音频做不输出正文的联合回归。                                             |
| Hedged requests    | 自用 fork 保留默认 `2 eager + 5s delayed`，但提供两个开关与 1–30 秒 delay 设置。官方 Discussion #1953 已被维护者明确拒绝，不创建代码 PR；只有维护者未来主动改变方向或要求实现时才重新评估。                      |
| Shared HTTP client | 已从干净 `upstream/main` 提炼到 `perf/reuse-post-processing-http-client`：实现 `11663a5`，请求级认证回归 `6649ce7`，mock 读取稳定性 `400777f`。它只定位为连接池复用和 3 秒建连边界，不宣称解决几十秒生成尾延迟。 |
| 历史重新后处理     | 先与 Discussion #826 / 已有 PR #851 的作者和维护者协调，说明本 fork 的无 migration/无版本历史最小子集。未获得方向前不创建竞争性 PR。                                                                             |

本地协调稿位于被 `.git/info/exclude` 排除的 `upstream-pr-drafts/`；它们不会进入产品 commit。官方 PR template 的 `Human Written Description` 必须由用户本人用 2–3 句自己的话填写，AI 只能留 TODO，不得代写或伪装成人工描述。

### 3.2 Hedged requests 上游追踪

追踪对象：[Handy Discussion #1953](https://github.com/cjpais/Handy/discussions/1953)。

当前结论：`rejected by maintainer`。不提交对应代码 PR，也不通过改名、拆分或直接开 Draft PR 绕过该结论。用户已在维护者评论下补充中国地区使用 DeepSeek、MiMo、通义千问时频繁出现 20–30 秒后处理尾延迟、采用 hedging 后不再遇到的实际情况，并强调只是记录问题、寻找相同体验，而非坚持合入。该补充没有推翻维护者决定；后续同步 `upstream/main`、收到该 Discussion 新通知或调整本 fork 的 hedging 策略时复查。只有维护者明确改变方向或主动要求实现，才重新开启上游贡献工作。

| 检查时间                      | Upvotes | 顶层评论 / 回复 | 结论与下一步                                                                                                                                                    |
| ----------------------------- | ------- | --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-08-23 19:38（+08:00）    | 1       | 1 / 0           | `cjpais` 明确拒绝合入，理由为成本、UI/认知复杂度及项目转向本地模型。保留 fork-only 实现，不创建代码 PR；继续记录后续是否出现方向反转。                          |
| 2026-08-23 19:41（+08:00）    | 1       | 1 / 0           | 无新增评论、回复或 reaction；Discussion 仍 open 且未锁定，维护者结论未变。继续保持 fork-only，不创建代码 PR。                                                   |
| 2026-08-23 19:43（+08:00）    | 1       | 1 / 0           | 应用户查看请求即时复查：评论者仍只有维护者 `cjpais`，维护者结论未变，不创建代码 PR。                                                                            |
| 2026-08-23 19:45（+08:00）    | 1       | 1 / 0           | 无新增评论、回复或 reaction；Discussion 的 `updatedAt` 仍为维护者原评论时间。继续保持 fork-only，不创建代码 PR。                                                |
| 2026-08-23 19:47（+08:00）    | 1       | 1 / 0           | 最终高频复查仍无变化。维护者决定已足以定稿：不创建代码 PR；后续仅在新通知、upstream 同步或本地策略调整时事件触发复查。                                          |
| 2026-08-23 19:49（+08:00）    | 1       | 1 / 1           | 用户以 `luweiCN` [补充真实使用场景](https://github.com/cjpais/Handy/discussions/1953#discussioncomment-18123668)；等待是否有进一步回复，现阶段仍不创建代码 PR。 |
| 2026-08-23 19:51（+08:00）    | 1       | 1 / 1           | 用户补充尚无新回复或 reaction，`updatedAt` 仍为该回复发布时间。继续等待事件触发反馈，维护者原拒绝和“不创建 PR”结论不变。                                        |
| 2026-08-23 19:53（+08:00）    | 1       | 1 / 1           | 仍无新回复或 reaction，Discussion 继续 open 且未锁定。没有重新评估上游实现的信号，不创建代码 PR。                                                               |
| 2026-08-23 19:53:58（+08:00） | 1       | 1 / 1           | 补充回复后的第三次即时复查仍无变化。结束分钟级轮询，后续按新通知、upstream 同步或本地策略调整事件触发；不创建代码 PR。                                          |

## 4. 合并冲突热点

| 文件                                                  | 必须核对的行为                                                                                                                                                    |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src-tauri/src/managers/transcription.rs`             | Qwen-only 30 秒 gate、25–30 秒静音切段、完整拼接、语言结果合并、错误不返回部分文本                                                                                |
| `src-tauri/src/llm_client.rs`                         | shared client、provider-specific reasoning、可配置 `1/2 eager + 0/1 delayed`（默认 `0s:2 + 5s:1`）、无 chat deadline、3 秒 connect timeout、8 秒 metadata timeout |
| `src-tauri/src/actions.rs`                            | `complete_unless_cancelled` 可被历史命令安全复用；普通转录取消仍能 drop LLM future                                                                                |
| `src-tauri/src/commands/history.rs`                   | 新命令始终读取 raw `transcription_text`，取消/失败不写数据库                                                                                                      |
| `src-tauri/src/managers/history.rs`                   | 只更新 LLM-derived 字段，保留原文，并发出既有 `Updated` event                                                                                                     |
| `src-tauri/src/lib.rs`                                | Tauri/Specta 命令注册仍包含 `retry_history_entry_post_process`                                                                                                    |
| `src/bindings.ts`                                     | tauri-specta 生成结果与 Rust 命令签名一致，不手工维护漂移版本                                                                                                     |
| `src/components/settings/history/HistorySettings.tsx` | 结果/原文展示、逐条 loading、取消、toast、键盘和 accessibility 语义                                                                                               |
| `src/i18n/locales/*/translation.json`                 | 全部 locale key 一致                                                                                                                                              |
| `src-tauri/src/catalog/catalog.json`                  | Qwen3-ASR 0.6B/1.7B 保持 `streaming: false`，除非 backend 已有真正原生实现                                                                                        |

如果上游已经实现等价功能，优先评估并采用上游版本以缩小长期 diff，但只有在上述不变量和测试全部通过后，才能删除本地重复实现。

## 5. 定期合并 `upstream/main`

### 5.1 合并前检查

在 Handy 仓库中执行：

```bash
git status --short --branch
git branch --show-current
git remote -v
git symbolic-ref --short refs/remotes/upstream/HEAD
git rev-parse HEAD
git rev-parse origin/feature/long-dictation-stable-postprocess
```

要求：

- 当前分支是 `feature/long-dictation-stable-postprocess`。
- `origin` 指向用户 fork，`upstream` 指向官方仓库。
- 上游默认分支仍是 `upstream/main`；若已改变，先查明官方迁移方式，不要盲目合并旧引用。
- 本地 HEAD 与 `origin/feature/long-dictation-stable-postprocess` 应一致。若不一致，先审阅双方提交并解释差异，不得 force-push。
- 工作树必须干净。若存在不属于本次任务的改动，停止并说明，不得自行 `stash`、commit、覆盖或删除。
- 不读取、打印或提交 `settings_store.json`、API key、历史数据库、录音或转录正文。

### 5.2 获取并审阅上游变化

```bash
git fetch --prune origin
git fetch --prune upstream
git log --oneline --decorate HEAD..upstream/main
git diff --stat HEAD...upstream/main
git diff --name-only HEAD...upstream/main
```

先阅读 release notes、dependency/DB migration、Tauri 权限、updater 和上述冲突热点的变化。不要看到可 fast-forward 或无文本冲突就直接假设行为兼容。

### 5.3 创建恢复点并合并

用实际时间替换占位符，先创建本地恢复分支：

```bash
git branch backup/custom-before-upstream-YYYYMMDD-HHMMSS
git merge --no-ff upstream/main
```

这里刻意使用 merge 而不是 rebase：自定义提交已经推送，merge 可以保留历史并避免 force-push。

出现冲突时：

1. 用 `git status` 和 `git diff --name-only --diff-filter=U` 列出冲突。
2. 逐文件理解上游意图和本地不变量；不要整批使用 `--ours` 或 `--theirs`。
3. 解决后运行 `git diff --check`，再执行第 6 节全部验证。
4. 如果无法可靠解决，在合并前工作树确认干净的前提下可以 `git merge --abort`，并向用户报告；不得用 `git reset --hard`。

如果 merge 已提交或已推送后才发现问题，优先从备份分支重新构建，或使用可审计的 `git revert -m 1 <merge-commit>`；不要重写公开分支历史。

## 6. 验证门槛

### 6.1 依赖与静态检查

如果 lockfile 或依赖声明变化，先执行：

```bash
bun install --frozen-lockfile
```

每次合并至少执行：

```bash
bun run format:check
bun run lint
bun run check:translations
bun run build
cargo test --manifest-path src-tauri/Cargo.toml --lib
cargo clippy --manifest-path src-tauri/Cargo.toml --lib --all-features
```

2026-08-22 的已知基线是 Rust tests `222/222`、LLM client tests `19/19`，前端 format/lint/translation/build 全部通过。标准 Clippy exit 0，但当时仍打印 11 个上游既有 warning；`-D warnings` 会因此失败。未来应区分既有 warning 与本次新增 warning，不要借合并之机顺手重构无关代码。

### 6.2 自定义行为的 targeted tests

```bash
cargo test --manifest-path src-tauri/Cargo.toml --lib qwen_long_audio
cargo test --manifest-path src-tauri/Cargo.toml --lib llm_client::tests::
cargo test --manifest-path src-tauri/Cargo.toml --lib update_post_processing_preserves_original_transcription
cargo test --manifest-path src-tauri/Cargo.toml --lib actions::tests::pending_operation_stops_after_cancellation
cargo test --manifest-path src-tauri/Cargo.toml --lib actions::tests::completed_operation_returns_its_output
```

还要人工检查：

- Qwen catalog 仍为 `streaming: false`。
- AppSettings 的 hedging defaults 是 parallel=true、delayed=true、delay=5 秒，delay command/UI 范围是 1–30 秒。
- mock tests 覆盖 1 only、2 eager、1 + 1 delayed、2 + 1 delayed 四种组合，首个非空成功结果仍唯一获胜。
- chat completion 没有整体 response timeout；model-list metadata 仍有独立 timeout。
- 历史 UI 的“重新后处理”仍以原文为输入，成功只更新 LLM 字段，取消可用。

不要在自动验证中调用真实 LLM API。相关 Rust 测试使用本地 mock server；真实调用可能产生三次费用并发送私人文本，必须得到用户明确许可。

### 6.3 私密长音频回归

只有本机已有、用户允许使用的测试音频时才执行。不要提交音频，不要显示或保存转录正文。使用明确的绝对路径设置任务专用变量：

```bash
HANDY_REGRESSION_AUDIO='/absolute/path/to/private-regression.wav'
HANDY_QWEN_MODEL='/absolute/path/to/Qwen3-ASR-1.7B-Q5_K_M.gguf'
set -o pipefail
src-tauri/target/release/bundle/macos/Handy\ Fork.app/Contents/MacOS/handy \
  --transcribe-file "$HANDY_REGRESSION_AUDIO" \
  --model "$HANDY_QWEN_MODEL" \
  --json 2>&1 >/dev/null |
  awk '/Qwen long-audio|status 18|output truncated|failed|splitting/'
```

已知 74.61 秒私密样本的安全基线是切为 3 段、进程 exit 0、没有 `status 18`；本文档刻意不记录其路径、文件名或正文。

## 7. macOS release 构建与检查

fork 使用独立的 Tauri minisign updater trust chain；它不是 Apple Developer 证书。内置 public key 与 `luweiCN/Handy` release endpoint 都在 `src-tauri/tauri.conf.json`，encrypted private key 只保存在本机受限文件和 GitHub Actions secret，password 只保存在 macOS Keychain 与对应 GitHub secret。不要重新生成、提交或输出 private key/password。

本地构建时从受限文件和 Keychain 注入子进程环境；不要把 secret 放进命令参数，也不要打开 shell tracing：

```bash
handy_fork_key_file='/Users/luwei/Library/Application Support/Handy Fork Updater/handy-updater.key'
handy_fork_private_key="$(< "$handy_fork_key_file")"
handy_fork_key_password="$(security find-generic-password -a luweiCN -s com.luweicn.handy.updater -w)"
TAURI_SIGNING_PRIVATE_KEY="$handy_fork_private_key" \
  TAURI_SIGNING_PRIVATE_KEY_PASSWORD="$handy_fork_key_password" \
  bun run tauri build -b app,dmg
unset handy_fork_private_key handy_fork_key_password
```

默认 artifact 位置：

- `src-tauri/target/release/bundle/macos/Handy Fork.app`
- `src-tauri/target/release/bundle/dmg/Handy Fork_<version>_aarch64.dmg`
- `src-tauri/target/release/bundle/macos/Handy Fork.app.tar.gz`
- `src-tauri/target/release/bundle/macos/Handy Fork.app.tar.gz.sig`

检查示例：

```bash
handy_fork_app='src-tauri/target/release/bundle/macos/Handy Fork.app'
file "$handy_fork_app/Contents/MacOS/handy"
codesign --verify --deep --strict "$handy_fork_app"
codesign -dv --verbose=2 "$handy_fork_app"
plutil -extract CFBundleIdentifier raw "$handy_fork_app/Contents/Info.plist"
plutil -extract CFBundleShortVersionString raw "$handy_fork_app/Contents/Info.plist"
shasum -a 256 "$handy_fork_app/Contents/MacOS/handy"
shasum -a 256 src-tauri/target/release/bundle/dmg/*.dmg
```

预期 architecture 为 `arm64`、bundle id 为 `com.luweicn.handyfork`、签名为 ad-hoc，archive 根目录为 `Handy Fork.app/`。ad-hoc 构建不是 notarized 官方发行包，因此 macOS 首次启动和隐私权限仍需用户手动确认。

公开 release 只能由 `.github/workflows/release-custom-macos.yml` 从产品分支创建。workflow 必须先创建 draft，并锁定 `Handy Fork`、`com.luweicn.handyfork`、fork endpoint 和 updater public-key fingerprint；核对 CI 产物、`latest.json`、GitHub digest、架构、bundle/version 和签名后才能发布为 Latest。不要运行官方多平台 release workflow。

## 8. 安装与回滚

安装是有状态操作。未来 AI 只有在用户明确要求安装时才能执行，并应遵循以下顺序：

1. 记录新 fork app binary hash、版本、bundle id 和签名检查结果。
2. 在 fork 身份首次迁移公开前，正常退出当前旧身份 fork，确认 `com.pais.handy` 进程已结束；避免它看到跨 bundle 的新 update。
3. 在 `~/Applications/Handy Backups/<timestamp>/` 创建明确、唯一的目录，完整备份当前 `/Applications/Handy.app` 和旧 `~/Library/Application Support/com.pais.handy` 数据目录，不覆盖既有备份。
4. 不读取或输出 settings、API key、history、录音或转录正文。备份只保留本机，不能提交或上传。
5. 首次迁移时将整个旧数据目录复制到 `~/Library/Application Support/com.luweicn.handyfork`，不得移动或删除原目录；若新目录已存在，先备份并停止，不要静默合并两个状态树。
6. 用 `ditto` 将已验收 fork app 复制到 `/Applications/Handy Fork.app`，绝不覆盖 `/Applications/Handy.app`。
7. 重新计算 installed fork binary hash 并复查 `Handy Fork` / `com.luweicn.handyfork` / version / arm64 / codesign / updater endpoint。
8. 官方 app 只能从 `cjpais/Handy` 官方 release 恢复到 `/Applications/Handy.app`，并核对官方 bundle id `com.pais.handy`；不要用 fork release 假冒官方 app。
9. 必要时执行 `tccutil reset Accessibility com.luweicn.handyfork`，再启动 fork。告知用户在 macOS“系统设置 → 隐私与安全性”手动允许 `Handy Fork` 的 Accessibility 和麦克风；AI 不代替用户批准系统隐私权限。
10. 只做必要的非敏感检查：两个 app 路径/bundle id、两个数据目录、两个 updater channel 和 fork 原有模型/历史数量。不要打印完整 settings 或 history 内容。

两个 bundle 的数据目录从首次复制后彼此独立；后续写入不会自动同步。两个 app 可以同时安装和分别更新，但相同 global shortcut 会冲突，用户应设置不同快捷键或避免同时运行。若 fork 迁移失败，先保留失败 app/新数据，再恢复备份的旧身份 fork；只有发现 settings/DB migration 不兼容时才恢复对应数据备份。

2026-08-25 独立身份本地构建快照（CI release 和安装完成后再补充 release digest/备份位置）：

| 项目            | SHA-256 / 位置                                                     |
| --------------- | ------------------------------------------------------------------ |
| app binary      | `f7428bbde7afe5e364da6297627089e49114ca36679ca88256857b976c1a822b` |
| DMG             | `7a949f22121a7ac5b08d3581288bc73f630989937fc3b95d1b265363b9a22ceb` |
| updater archive | `b41d970ca134165c054b00ec4fe5c64f6c3401aa956209538c4b57cb9a2d729f` |
| updater `.sig`  | `270bb83ee3bfc029399faf67386d303cb2d3d5a93e77a41cad08263f7dd6769e` |
| 待安装路径      | `/Applications/Handy Fork.app`                                     |
| 待迁移数据目录  | `~/Library/Application Support/com.luweicn.handyfork`              |

这些 hash 只用于识别 2026-08-25 的本地构建。下一次合并或 CI 重建后 hash 变化是正常的，必须记录对应产物的新值，而不是要求继续匹配旧值。

## 9. 提交、推送与维护记录

所有验证和安装检查通过后：

```bash
git status --short
git log --oneline --decorate --max-count=15
git push origin feature/long-dictation-stable-postprocess
```

- 不使用 force-push。
- 不自动向 upstream 创建 PR。
- 将本次上游基线、merge commit、冲突、测试结果、artifact hash、备份位置和 Accessibility 状态更新到本文档。
- 文档更新可使用独立提交，例如 `docs: update custom fork maintenance snapshot`。

维护记录：

| 日期       | 上游基线  | 自定义功能基线 | 结果                                                                                  |
| ---------- | --------- | -------------- | ------------------------------------------------------------------------------------- |
| 2026-08-22 | `0e50367` | `1881d5a`      | 完成初始自定义功能、验证、打包和安装；等待用户重新授予 Accessibility                  |
| 2026-08-23 | `0e50367` | `1881d5a`      | 发布 opt-in hedging Discussion #1953；维护者明确拒绝合入，自用行为不变且不创建代码 PR |
| 2026-08-25 | `20ada47` | `3292f1e`      | 合入 v0.9.6 后 main，建立 fork updater；开始迁移为独立 app/bundle/update channel      |

## 10. 未来 AI 的完成报告

每次维护结束必须向用户明确报告：

- 合入了哪些 upstream commits，最终 branch HEAD 是什么。
- 哪些文件发生冲突，如何保留了第 2 节的不变量。
- full/targeted tests、format、lint、translations、build 和 Clippy 的实际结果。
- 是否运行了真实长音频或真实 LLM；若没有，明确说明。
- 新 app/DMG 的版本、architecture、签名和 SHA-256。
- 安装前备份的明确位置、安装是否成功、如何回滚。
- Accessibility 是否需要用户重新授权。
- 是否已推送到 `origin`，本地与远端 HEAD 是否一致。

## 11. 可直接交给未来 AI 的提示词

```text
请先完整阅读 AGENTS.md 和 CUSTOM_FORK_MAINTENANCE.md。检查 Handy 仓库工作树、当前分支和两个 remote；如果存在非本次改动，停止并报告，不要 stash、覆盖或删除。获取 origin 和 upstream 最新引用，审阅 upstream/main 相对当前分支的变化，创建 timestamped 本地备份分支，再用 merge（不要 rebase/force-push）把 upstream/main 合入 feature/long-dictation-stable-postprocess。

解决冲突时必须保留维护文档第 2 节的产品不变量：Qwen 长音频静音感知切段；不恢复软件伪 streaming；所有远程后处理默认 t=0 双发、5 秒 delayed third，同时保留两个独立开关与 1–30 秒 delay 设置；任何组合都以首个非空成功结果唯一获胜，chat 无自动截止但可手动取消；历史重新后处理始终使用原始 transcription_text 且失败/取消不覆盖旧结果。若上游已有等价实现，可采用上游版本，但需用测试证明行为等价。

完成 full/targeted tests、前端检查、translations、Clippy、macOS app/DMG 构建、签名/hash 和隐私安全的长音频回归。未经我明确许可，不调用真实 LLM API，不读取/输出 API key、录音或转录正文。只有我明确要求安装时，才先备份当前 app、settings 和 history，再安装并验证；最后更新维护文档、提交并正常 push 到 origin，给出完整结果和回滚位置。
```
