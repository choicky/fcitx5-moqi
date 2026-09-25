# Architecture Decisions

本文件只记录已经接受的重要技术决策。决策可随源码研究和 PoC 结果被明确修订或替代，但不得无记录改变。

## D001 — Android 为第一目标平台

**状态：Accepted**

优先完成 Android 输入体验和架构验证。其他平台后续研究。

## D002 — Pinyin/Shuangpin 为主，MoQi 为辅助码

**状态：Accepted**

正常输入由 Pinyin/Shuangpin、词库和语言模型完成；MoQi 仅在需要消除歧义时按需筛选候选。

## D003 — 采用早期墨奇逐字/词交互

**状态：Accepted**

不复制新版“输入完整句子后通过句中任意辅助码选择并立即提交整句”的模式。MoQi 不应天然触发整句 commit，过滤/选择后应能继续 composition 并再次使用辅码。

## D004 — 将现有 Stroke Filter 最小泛化为 Auxiliary Filter 基础设施

**状态：Accepted**

复用 `fcitx5-chinese-addons` 已有 Stroke Filter 的 trigger handling、mode lifecycle、CandidateList filter、Backspace/Escape、selection 和 composition 协作。

不继续维护独立平行的 Stroke/MoQi trigger 与 mode 状态机。Stroke-specific 与 MoQi-specific filtering algorithm 保持独立。

## D005 — Auxiliary Filter Trigger 与 Filter 实现解耦

**状态：Accepted**

反引号 `` ` `` 为默认 Auxiliary Filter Trigger，仅表达“进入当前配置的 Filter”。

Configured Auxiliary Filter 至少支持：

- Disabled
- Stroke
- MoQi

Trigger 不得硬编码为 Stroke 或 MoQi。

## D006 — MoQi 使用 selection-frontier target semantics

**状态：Accepted**

MoQi V1 过滤 current selection frontier 后的目标字符，不复制 Stroke 当前“候选 phrase 中任意字符匹配即可保留”的语义。

## D007 — MoQi V1 复用 LibIME partial selection，不修改 LibIME

**状态：Accepted for V1**

复用 `selectedLength()`、`candidatesToCursor()`、`selectCandidatesToCursor()` 等现有能力保留 composition。只有 Phase 2 PoC 证明上层接口不足时才重新评估 LibIME 修改。

## D008 — 固定 MoQi table 来源和版本

**状态：Accepted**

使用 `gaboolic/moqima-tables`，固定 commit `6d8ba8f1c57466f358e682baefe11bbd0fe389ab`，当前使用 `moqima_gb18030.txt`。测试不得猜测 MoQi code。

## D009 — Rime 是参考与备选，不是硬依赖

**状态：Accepted**

当前优先 Fcitx5 Pinyin/Shuangpin + LibIME + Auxiliary Filter；Rime/rime-frost 保留为成熟参考和 fallback。

## D010 — 优先复用 Android generic Fcitx configuration UI

**状态：Accepted**

Auxiliary Filter selection 优先通过 Fcitx config descriptor 暴露，复用 `fcitx5-android` 现有 ConfigEnum/ConfigKey UI。除非验证不足，不增加 MoQi-specific settings UI。

## D011 — 词库更新与用户数据上传解耦

**状态：Accepted**

允许联网更新词库，但不得把词库更新与上传用户输入历史绑定。

## D012 — 默认 Voice Trigger 为麦克风按钮

**状态：Accepted**

独立麦克风按钮为默认 Voice Trigger。Trigger 只表示开始/进入语音输入，不绑定任何 ASR Provider。

## D013 — 长按 Space 可选触发同一个 Voice Input

**状态：Accepted**

后续在 `SpaceLongPressBehavior` 增加 `VoiceInput`。麦克风按钮与长按 Space 必须 dispatch 到同一 Voice Trigger，不建立两套 voice pipeline。

## D014 — 优先复用 Fcitx5 Android SpeechRecognizer 工作

**状态：Accepted**

未来 Voice PoC 先研究和复用 upstream Fcitx5 Android 已有麦克风能力及 WIP SpeechRecognizer voice-input 工作，不从零重写 Android speech client。

## D015 — SpeechRecognizer / RecognitionService 为优先 Android speech boundary

**状态：Accepted**

优先采用：

`Fcitx5 Android -> SpeechRecognizer -> RecognitionService`

其中 RecognitionService 是 Android speech implementation 标准边界，不等同于项目内部 ASR Provider abstraction。

## D016 — ASR Provider 独立可插拔

**状态：Accepted**

RecognitionService / Voice Service 后保持独立 ASR Provider 层。更换云端、本地、自建或 OpenAI-compatible Provider 不改变基本 Voice Trigger 交互。

## D017 — ASR 与 LLM 后处理解耦

**状态：Accepted**

ASR 只负责 Audio → Raw Transcript。LLM/Text Post Processor 独立、可关闭，ASR Provider 与 LLM Provider 分别配置。

## D018 — Voice 数据流必须透明可审计

**状态：Accepted**

必须能够确定录音开始/停止、ASR Provider、endpoint、上传数据、Raw Transcript、是否进入 LLM、LLM endpoint 和最终提交文本。

## D019 — 总仓库与上游 fork 分离

**状态：Accepted**

`fcitx5-moqi` 为总控仓库。Phase 2 使用 `choicky/fcitx5-chinese-addons` fork。当前不 fork LibIME；是否 fork `fcitx5-android` 待 Voice PoC 实际修改边界确认。

## D020 — 暂不确定总仓库 LICENSE

**状态：Accepted**

在确认未来纳入代码和上游/码表再分发边界前，不急于选择总仓库 LICENSE。

## D021 — 最小修改、不过度抽象

**状态：Accepted**

Phase 2 不为未来 Filter 建立复杂 Plugin Framework；优先在 `fcitx5-chinese-addons` 完成 MoQi V1，不修改 LibIME 或 Android candidate protocol，除非 PoC 证明必要。

## D022 — CI 采用批量验证策略

**状态：Accepted**

相关改动组成逻辑完整、可审查的批次。push 前先完成源码/API 核对、diff review、格式/静态检查和适用本地测试。GitHub Actions 用于阶段性集成验证，不作为猜测性试错工具；纯文档修改原则上不触发重型 CI。

## D023 — Auxiliary Filter 配置持久化沿用上游路径，单元测试不覆盖

**状态：Accepted**

`AuxiliaryFilter` 是 `PinyinEngineConfig` 的普通 Enum option，沿用上游 `InputMethodEngine::setConfigForInputMethod()` → `PinyinEngine::setConfig()` / `reloadConfig()` 的既有持久化路径；本 fork 未修改这些函数，不新增 fork 面，也不为持久化改造测试框架。

单元测试只验证 Android generic config 契约中可在进程内验证的部分：descriptor 暴露（Type / DefaultValue / Enum / EnumI18n）与 `setConfigForInputMethod()` → `getConfigForInputMethod()` 三个配置值往返。

磁盘持久化与 reload 后取值必须在 Android 实机验证：测试环境以 `SkipUserPath` 构造 `StandardPaths`，`userPath(PkgConfig)` 为空，`safeSaveAsIni()` 无处可写、`readAsIni()` 读不到文件，`Configuration::load()` 会把所有选项 reset 为默认值。
