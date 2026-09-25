# Architecture Decisions

本文件记录已经形成的重要技术决策。决策可以随着源码研究和 PoC 结果被修订或替代，但不应无记录地改变。

## D001 — Android 为第一目标平台

**状态：Accepted**

优先完成 Android 上的输入体验和架构验证。Windows 等平台后续再研究，避免初期同时解决多平台问题。

## D002 — 拼音/双拼为主，墨奇为辅助码

**状态：Accepted**

墨奇不作为主要输入编码。正常输入由 Pinyin/Shuangpin、词库和语言模型完成；仅在需要消除同音歧义时使用墨奇辅助筛选。

## D003 — 偏好早期墨奇交互，不复制新版“句中任意辅助码”

**状态：Accepted**

目标是逐字/词、按需使用辅码。输入辅码不应天然触发整句 commit。完成一个字的筛选后，应尽量能够继续处理其他字词。

理由：新版“先输入完整句子拼音，再通过句中任意辅助码选择整句并立即上屏”的方式，在候选句仍含错字或句中拼音输错时，局部纠错体验不符合本项目目标。

## D004 — 优先复用 Fcitx5 已有辅助筛选机制

**状态：Accepted for V1**

优先研究 `fcitx5-chinese-addons` Pinyin 已有 Stroke Filter，并尝试以新增 MoQi Filter 的方式实现，而不是重新设计辅助码系统。

目标是保留 Stroke Filter，并新增墨奇筛选能力。

源码研究已确认 Stroke Filter 在 `fcitx5-chinese-addons` 候选层通过 `CommonCandidateList::setFilter()` 工作，无需进入 LibIME decoder。MoQi V1 将优先复用该框架。

注意：现有 Stroke Filter 对多字候选采用“任意字符匹配即保留”的语义，这不直接等同于本项目希望的逐字/词墨奇交互。目标字/词的约束语义仍需通过 partial selection 与 `ChooseCharFromPhrase` 的后续研究确定。

## D005 — 尽量不修改 LibIME 核心

**状态：Provisional**

LibIME 继续负责拼音解码、词典、Language Model 和用户学习。MoQi Filter 原则上应位于更上层的候选筛选逻辑。

只有源码研究证明现有接口不足以实现目标交互时，才考虑修改 LibIME。

## D006 — Rime 是参考与备选，不是硬依赖

**状态：Accepted**

Rime/rime-frost + 墨奇已经提供成熟参考，并可作为 fallback。当前优先研究 Fcitx5 原生 Pinyin/Shuangpin + LibIME + MoQi Filter，以评估能否得到更直接、维护边界更清晰的实现。

## D007 — 词库更新与用户数据上传解耦

**状态：Accepted**

允许联网下载/更新词库，但不能把“联网更新词库”与“上传用户输入数据”绑定为同一机制。

## D008 — ASR 与 LLM 后处理解耦

**状态：Accepted**

ASR 负责音频到原始文本；LLM 仅作为可选文本后处理阶段。两者 provider 可以独立配置，LLM 可以完全关闭。

## D009 — 语音不要求完全离线，但数据流必须透明

**状态：Accepted**

允许云端、本地和自建 ASR。必须能够明确录音、上传目标、上传内容、停止条件，以及是否进行后续 LLM 处理。

## D010 — 当前总仓库不等同于上游 fork

**状态：Accepted**

`fcitx5-moqi` 当前作为需求、研究、设计和集成工作的总控仓库。

暂不 fork：

- `fcitx5-android`
- `fcitx5-chinese-addons`
- LibIME

待 Phase 1 确认实际修改边界后，再精准 fork 必须修改的上游项目。当前最可能需要 fork 的候选是 `fcitx5-chinese-addons`，但尚未定案。

## D011 — 暂不确定本仓库 LICENSE

**状态：Accepted**

在确认未来纳入本仓库的代码、上游许可证及墨奇码表的再分发边界前，不急于选择总仓库 LICENSE。后续在进入代码实现阶段前重新评估。


## D012 — Auxiliary Filter 抽象，当前 MoQi first

**状态：Accepted**

拼音/双拼继续由 LibIME 产生候选，上层通过 Auxiliary Filter 进行按需候选过滤。当前实现和验证只聚焦 MoQi Filter；Radical、Stroke 等作为未来可扩展 Filter，不为尚未实施的功能过度设计。

MoQi Filter 不应强制 commit 整句；过滤后应尽量保留 composition，使用户能够撤销辅码、继续输入，并对其他字/词再次筛选。

## D013 — ASR Provider 从一开始可插拔

**状态：Accepted**

语音链路采用：

`Audio Capture -> ASR Provider -> Raw Transcript -> Optional LLM Post Processor -> IME`

不绑定单一厂商或模型。最终由用户在 Fcitx5 Android UI 中选择和配置 ASR Provider。云端、本地、自建和 OpenAI-compatible Provider 均可通过同一抽象接入；ASR 与 LLM 后处理继续保持独立。

## D014 — CI 采用批量验证策略

**状态：Accepted**

相关改动尽量组织成逻辑完整的批次，再统一 push 并触发 CI，避免“一个小改动 -> push -> 等待 Actions -> 再改”的循环。

本地可完成的检查优先本地执行；纯文档修改原则上不应触发耗时构建。CI 主要用于阶段性集成验证，可并行的验证尽量一次触发。批量修改仍应保持目标明确、规模可审查。
