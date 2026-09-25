# Roadmap

> 路线图按当前已验证架构安排；源码研究或 PoC 结果可以触发有记录的调整。

## Phase 0 — 需求确认

**状态：COMPLETE**

已明确 Android first、Pinyin/Shuangpin 主输入、MoQi Auxiliary Filter、早期逐字/词交互、Trigger/Implementation 解耦、Voice Trigger、ASR/LLM 解耦及隐私边界。

## Phase 1 — 上游源码与架构研究

**状态：COMPLETE**

已完成 Phase 2 所需关键研究：

- Stroke Filter → CandidateList → composition 调用链；
- `CommonCandidateList::setFilter()`；
- LibIME partial selection / `selectedLength()` / `candidatesToCursor()` / `selectCandidatesToCursor()`；
- Android TabbedCandidateList plumbing；
- Android generic Fcitx config UI；
- MoQi target semantics；
- MoQi table 来源、版本、许可证和实际数据验证；
- MoQi reverse lookup foundation；
- Fcitx5 Android 现有 voice UI 与 upstream SpeechRecognizer 工作；
- Android SpeechRecognizer / RecognitionService 边界。

结论：Phase 2 优先只修改 `fcitx5-chinese-addons`，当前无需修改 LibIME 或 Android candidate protocol。

## Phase 2 — MoQi Auxiliary Filter V1 PoC

**状态：IN PROGRESS**

### 目标

把此前平行的 MoQi/Stroke mode 设计重构为基于上游 Stroke Filter 的统一 Auxiliary Filter，并完成真实端到端 PoC。

### 当前批次

- [x] 将现有 Stroke trigger/mode 最小泛化为 Auxiliary Filter；
- [x] 增加 Disabled / Stroke / MoQi 配置；
- [x] 保留 Stroke-specific filter；
- [x] 接入现有 MoQi reverse lookup；
- [x] 实现 selection-frontier MoQi filtering；
- [x] Backspace / Escape；
- [x] partial selection；
- [x] composition 保留；
- [x] 继续输入；
- [x] 再次使用 Auxiliary Filter；
- [x] Pinyin / Shuangpin 等价核心行为；
- [x] Stroke regression；
- [ ] 验证 generic Android config exposure（仅完成进程内契约验证，实机验证未做）。

实现已提交到 `choicky/fcitx5-chinese-addons` 的 `feature/moqi-filter` 分支，当前 tip 为 `f903176f8ffe970bd9e4baf3d974d6b3828c85c5`，对应 PR #1。源码已包含上述已勾选能力及 Pinyin/Shuangpin 自动化测试。

CI（run `36121191299`，tip `f903176`）三个 job 全部通过：clang-format、Build and test (gcc)、Build and test (clang)；ctest 9/9 全部通过，其中 `testpinyinhelper` 用固定码表验证墨奇反查，`testpinyin` 覆盖 Stroke / MoQi / Disabled、partial selection、继续输入与 config 契约。

Stroke regression 的证据是上游既有测试 `testActionInStrokeFilter`、`testPinyinTabFilter`、`testPinyinTabFilterWithSeparator` 在重构后通过（仅新增显式设置 `AuxiliaryFilter=Stroke`，其余过滤流程未改）。

此前 `testAuxiliaryFilterConfigContract` 含一条在该测试环境下结构性无法成立的断言（`reloadConfig()` 之后按输入法配置取值），触发 `FCITX_ASSERT` 中止整个测试二进制，导致其后所有 Stroke / MoQi 测试从未执行。该断言已移除并在测试源码中注明原因。

Android generic config exposure 目前只有进程内验证：config descriptor 把 AuxiliaryFilter 暴露为 Enum（Type / DefaultValue / Enum[i] / EnumI18n[i]），且 `setConfigForInputMethod()` → `getConfigForInputMethod()` 对 Disabled / Stroke / MoQi 三个值往返一致。磁盘持久化与 reload 后的取值无法在单元测试框架内验证：测试环境以 `SkipUserPath` 构造 `StandardPaths`，`userPath(PkgConfig)` 为空，`safeSaveAsIni()` 无处可写、`readAsIni()` 读不到文件，`Configuration::load()` 于是把所有选项 reset 为默认值。该项需 Android 实机验证后才能勾选。

### 预计修改边界

主要：`fcitx5-chinese-addons`

当前不修改：

- LibIME；
- Android candidate frontend protocol；
- MoQi-specific Android UI。

### Exit Criteria

以下流程端到端成立：

```text
Pinyin/Shuangpin
→ candidates
→ Auxiliary Filter Trigger
→ configured MoQi
→ real MoQi code
→ candidate filtering
→ partial selection
→ composition preserved
→ continue input
→ Auxiliary Filter Trigger again
→ second filtering/selection
```

同时要求 Disabled、Stroke、Backspace、Escape 均正确，测试只使用固定真实 MoQi table 数据。

Phase 2 Exit Criteria 满足前，不进入完整 Android UI 或 Voice 实现。

## Phase 3 — Full MoQi / Android Integration

**状态：NOT STARTED**

Phase 2 成功后：

- 完整固定 MoQi table 集成；
- 最终 build-time transformation / version update policy；
- edge cases 与完整测试；
- Android 构建、安装和实际输入体验；
- Auxiliary Filter 配置体验；
- 用户学习、性能和稳定性；
- 评估向上游贡献及长期 fork 必要性。

## Phase 4 — Voice Input PoC

**状态：NOT STARTED**

优先复用：

- Fcitx5 Android 现有 microphone UI；
- upstream WIP SpeechRecognizer voice-input 工作；
- Android `SpeechRecognizer`；
- Android `RecognitionService`。

验证：

```text
Microphone
→ Voice Trigger
→ SpeechRecognizer
→ RecognitionService
→ ASR
→ Raw Transcript
→ IME
```

并验证：

```text
Long-press Space
→ same Voice Trigger
```

Exit Criteria：

- microphone 与可选 long-press Space 进入同一 voice path；
- permission/lifecycle/start/stop/cancel 正确；
- partial/final transcript 正确；
- Voice Trigger 不绑定 ASR vendor；
- ASR implementation boundary 明确；
- 数据流可审计。

## Phase 5 — ASR Provider Architecture / PoC

**状态：NOT STARTED**

根据 Phase 4 实际需求建立最小 Provider abstraction，验证代表性的：

- cloud ASR；
- OpenAI-compatible；
- local/self-hosted ASR；
- provider switching；
- streaming/non-streaming；
- 中文质量、延迟和数据流。

不在 Voice PoC 前过度设计 Provider framework。

## Phase 6 — Optional LLM Post-processing

**状态：NOT STARTED**

实现：

```text
ASR
→ Raw Transcript
→ Optional Text Post Processor
→ Final Transcript
```

要求可完全关闭、与 ASR Provider 独立、LLM Provider 独立配置，并明确发送文本和隐私边界。

## Phase 7 — Android Product Integration

**状态：NOT STARTED**

最终整合：

- Auxiliary Filter settings；
- Voice settings；
- ASR Provider settings；
- optional LLM settings；
- privacy/data-flow UI；
- packaging/release。

## Phase 8 — Additional Platforms

**状态：NOT STARTED**

Android 架构稳定后再评估 Windows、Linux、macOS、iOS，并保持 Trigger / Configured Implementation 分离。

## 工程节奏

- 修改前核对源码/API/现有测试；
- 逻辑完整的小批次开发；
- push 前 diff review、格式/静态检查和适用本地测试；
- GitHub Actions 仅作阶段性集成验证；
- 不通过反复提交猜测性修复；
- docs-only 原则上不触发重型 CI。

## 当前下一步

在 Android 实机上验证 Auxiliary Filter 配置项的显示、持久化与实际生效；这是 Phase 2 最后一个未勾选项，自动化测试无法替代。全部 Exit Criteria 通过后再将 Phase 2 标记为 COMPLETE。
