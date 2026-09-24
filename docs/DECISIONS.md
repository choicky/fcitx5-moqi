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

**状态：Provisional**

优先研究 `fcitx5-chinese-addons` Pinyin 已有 Stroke Filter，并尝试以新增 MoQi Filter 的方式实现，而不是重新设计辅助码系统。

目标是保留 Stroke Filter，并新增墨奇筛选能力。

在完整阅读 Stroke Filter 实现后重新确认本决策。

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
