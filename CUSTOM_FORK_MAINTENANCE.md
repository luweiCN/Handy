# Handy 自定义 Fork 维护手册

本文档记录 `luweiCN/Handy` 自定义分支的产品约束、实现差异，以及以后将 `cjpais/Handy` 最新主分支合并进来时应执行的流程。目标读者是未来接手仓库的 AI 或开发者。

> 先完整阅读仓库根目录的 `AGENTS.md` 和本文档，再修改、合并、构建或安装。代码和测试是运行事实；如果它们与本文档不一致，应先查明原因，而不是静默选择其中一个，并在验证后更新本文档。

## 1. 当前维护范围

截至 2026-08-22：

| 项目               | 当前值                                                 |
| ------------------ | ------------------------------------------------------ |
| 用户 fork          | `https://github.com/luweiCN/Handy`                     |
| `origin`           | `https://github.com/luweiCN/Handy.git`                 |
| `upstream`         | `https://github.com/cjpais/Handy.git`                  |
| 自定义分支         | `feature/long-dictation-stable-postprocess`            |
| 最后合入的上游基线 | `0e50367`，本地引用为 `upstream/main`                  |
| 自定义功能基线     | `1881d5a`                                              |
| macOS bundle       | `com.pais.handy`，版本 `0.9.5`，Apple Silicon `arm64`  |
| 本地签名           | ad-hoc；没有 notarization，也没有 Apple Team ID        |
| 当前 LLM 配置      | DashScope OpenAI-compatible endpoint + `qwen3.7-flash` |
| 官方自动更新       | 本机设置中已关闭，避免官方包覆盖自定义版               |

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

当前所有远程 provider/model 共用以下 hedged-request 策略。这里的 request 指一个 logical attempt；单个 attempt 内部还可能发生 reasoning-field 兼容重试。

1. `t=0` 向当前选中的同一 endpoint/model 同时发送两个相同请求。
2. 如果 5 秒内没有得到非空成功结果，发送第三个相同请求；即使前两个很早报错或返回空内容，第三个也不能提前到 5 秒之前。
3. 第一个非空成功结果获胜；错误或空响应不能抢占成功结果。
4. 获胜后在客户端 abort 其余任务。
5. chat completion 没有 response/generation deadline；供应商很慢时继续等待，由用户手动取消。

边界条件：

- 连接建立仍有 3 秒 `connect_timeout`，它不限制连接成功后的生成时间。
- `/models` metadata 请求单独保留 8 秒 timeout，因为该操作没有同等清晰的长期等待/取消体验。
- 客户端取消或 abort 不保证供应商停止服务端推理，也不保证停止计费。正常情况下，一次后处理应按最多三个可能完整生成/计费的 logical attempts 估算费用和配额。
- 未识别 endpoint 首次拒绝 `reasoning_effort` 时，每个并行 attempt 都可能先收到 400/422 再发一次无 reasoning 字段的兼容请求，因此物理 HTTP 请求数可能超过三个；被拒绝的探测通常未生成内容，但仍会消耗请求配额。
- 三次请求只发给用户当前选择的同一 endpoint/model，不自动把私人文本扩散到第二家供应商。
- 正常录音和历史重新后处理都必须使用现有 cancellation generation，确保用户取消后 drop 正在等待的 future。

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

## 4. 合并冲突热点

| 文件                                                  | 必须核对的行为                                                                                                                   |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `src-tauri/src/managers/transcription.rs`             | Qwen-only 30 秒 gate、25–30 秒静音切段、完整拼接、语言结果合并、错误不返回部分文本                                               |
| `src-tauri/src/llm_client.rs`                         | shared client、provider-specific reasoning、`0s:2 + 5s:1` hedging、无 chat deadline、3 秒 connect timeout、8 秒 metadata timeout |
| `src-tauri/src/actions.rs`                            | `complete_unless_cancelled` 可被历史命令安全复用；普通转录取消仍能 drop LLM future                                               |
| `src-tauri/src/commands/history.rs`                   | 新命令始终读取 raw `transcription_text`，取消/失败不写数据库                                                                     |
| `src-tauri/src/managers/history.rs`                   | 只更新 LLM-derived 字段，保留原文，并发出既有 `Updated` event                                                                    |
| `src-tauri/src/lib.rs`                                | Tauri/Specta 命令注册仍包含 `retry_history_entry_post_process`                                                                   |
| `src/bindings.ts`                                     | tauri-specta 生成结果与 Rust 命令签名一致，不手工维护漂移版本                                                                    |
| `src/components/settings/history/HistorySettings.tsx` | 结果/原文展示、逐条 loading、取消、toast、键盘和 accessibility 语义                                                              |
| `src/i18n/locales/*/translation.json`                 | 全部 locale key 一致                                                                                                             |
| `src-tauri/src/catalog/catalog.json`                  | Qwen3-ASR 0.6B/1.7B 保持 `streaming: false`，除非 backend 已有真正原生实现                                                       |

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
- `LLM_HEDGE_DELAY` 是 5 秒、`LLM_EAGER_ATTEMPTS` 是 2。
- chat completion 没有整体 response timeout；model-list metadata 仍有独立 timeout。
- 历史 UI 的“重新后处理”仍以原文为输入，成功只更新 LLM 字段，取消可用。

不要在自动验证中调用真实 LLM API。相关 Rust 测试使用本地 mock server；真实调用可能产生三次费用并发送私人文本，必须得到用户明确许可。

### 6.3 私密长音频回归

只有本机已有、用户允许使用的测试音频时才执行。不要提交音频，不要显示或保存转录正文。使用明确的绝对路径设置任务专用变量：

```bash
HANDY_REGRESSION_AUDIO='/absolute/path/to/private-regression.wav'
HANDY_QWEN_MODEL='/absolute/path/to/Qwen3-ASR-1.7B-Q5_K_M.gguf'
set -o pipefail
src-tauri/target/release/bundle/macos/Handy.app/Contents/MacOS/handy \
  --transcribe-file "$HANDY_REGRESSION_AUDIO" \
  --model "$HANDY_QWEN_MODEL" \
  --json 2>&1 >/dev/null |
  awk '/Qwen long-audio|status 18|output truncated|failed|splitting/'
```

已知 74.61 秒私密样本的安全基线是切为 3 段、进程 exit 0、没有 `status 18`；本文档刻意不记录其路径、文件名或正文。

## 7. macOS release 构建与检查

官方 updater public key 存在，但本地没有官方 updater private key。构建 app/DMG 时关闭 updater artifacts：

```bash
bun run tauri build -b app,dmg -c '{"bundle":{"createUpdaterArtifacts":false}}'
```

默认 artifact 位置：

- `src-tauri/target/release/bundle/macos/Handy.app`
- `src-tauri/target/release/bundle/dmg/Handy_<version>_aarch64.dmg`

检查示例：

```bash
file src-tauri/target/release/bundle/macos/Handy.app/Contents/MacOS/handy
codesign --verify --deep --strict src-tauri/target/release/bundle/macos/Handy.app
codesign -dv --verbose=2 src-tauri/target/release/bundle/macos/Handy.app
plutil -extract CFBundleIdentifier raw src-tauri/target/release/bundle/macos/Handy.app/Contents/Info.plist
plutil -extract CFBundleShortVersionString raw src-tauri/target/release/bundle/macos/Handy.app/Contents/Info.plist
shasum -a 256 src-tauri/target/release/bundle/macos/Handy.app/Contents/MacOS/handy
shasum -a 256 src-tauri/target/release/bundle/dmg/*.dmg
```

预期 architecture 为 `arm64`、bundle id 为 `com.pais.handy`、签名为 ad-hoc。ad-hoc 构建不是 notarized 官方发行包。

## 8. 安装与回滚

安装是有状态操作。未来 AI 只有在用户明确要求安装时才能执行，并应遵循以下顺序：

1. 记录新 app binary hash、版本、bundle id 和签名检查结果。
2. 正常退出 Handy，确认进程已结束。
3. 在 `~/Applications/Handy Backups/<timestamp>/` 创建明确、唯一的目录。
4. 将当前 `/Applications/Handy.app` 完整移动到该目录，不覆盖既有备份。
5. 将私密的 `settings_store.json` 和 `history.db` 复制到同一恢复目录；不得读取或输出内容。该备份含凭据和私人文本，只能保留在本机，不能提交或上传。模型和录音目录保持原位。
6. 用 `ditto` 将新 `.app` 复制到 `/Applications/Handy.app`。
7. 再次计算已安装 binary hash，必须与构建产物一致；复查签名和 bundle id。
8. 执行 `tccutil reset Accessibility com.pais.handy` 后启动应用。
9. 告知用户在 macOS“系统设置 → 隐私与安全性 → 辅助功能”重新允许 Handy。AI 不代替用户批准系统隐私权限。
10. 在 UI 中确认模型、provider、prompt、历史记录仍存在，官方自动更新仍关闭；不要通过打印完整设置文件来检查。

相同 bundle id 会复用 `~/Library/Application Support/com.pais.handy`，其中包含设置、历史、录音和模型。若需要回滚，先保留失败版本和当前数据，再恢复上一份 `.app`；只有发现 settings/DB migration 不兼容时才恢复对应数据备份。

2026-08-22 最终安装快照：

| 项目            | SHA-256 / 位置                                                                          |
| --------------- | --------------------------------------------------------------------------------------- |
| app binary      | `2605c65e63285a37349b9bf8c89a99fdda7cfc274e68eec465d6b744e042fd22`                      |
| DMG             | `0dfc3abc4e991b4af1fb785262f2bcc94a3467d0eecd84f65c7e592e22753f20`                      |
| 上一版 app 备份 | `~/Applications/Handy Backups/2026-08-22-164522/Handy-custom-pre-history-reprocess.app` |

这些 hash 只用于识别 2026-08-22 的构建。下一次合并后 hash 变化是正常的，必须记录新的构建值，而不是要求继续匹配旧值。

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

| 日期       | 上游基线  | 自定义功能基线 | 结果                                                                 |
| ---------- | --------- | -------------- | -------------------------------------------------------------------- |
| 2026-08-22 | `0e50367` | `1881d5a`      | 完成初始自定义功能、验证、打包和安装；等待用户重新授予 Accessibility |

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

解决冲突时必须保留维护文档第 2 节的产品不变量：Qwen 长音频静音感知切段；不恢复软件伪 streaming；所有远程后处理保持 t=0 双发、5 秒第三发、首个非空成功获胜、chat 无自动截止但可手动取消；历史重新后处理始终使用原始 transcription_text 且失败/取消不覆盖旧结果。若上游已有等价实现，可采用上游版本，但需用测试证明行为等价。

完成 full/targeted tests、前端检查、translations、Clippy、macOS app/DMG 构建、签名/hash 和隐私安全的长音频回归。未经我明确许可，不调用真实 LLM API，不读取/输出 API key、录音或转录正文。只有我明确要求安装时，才先备份当前 app、settings 和 history，再安装并验证；最后更新维护文档、提交并正常 push 到 origin，给出完整结果和回滚位置。
```
