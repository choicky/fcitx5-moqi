# fcitx5-moqi

面向 Android 的 Fcitx5 中文输入方案研究与实现项目。

## 目标

- 以 Fcitx5 Pinyin/Shuangpin + LibIME 为中文主输入；
- 墨奇码作为按需 Auxiliary Filter，而不是主输入编码；
- 复用并最小泛化 `fcitx5-chinese-addons` 现有 Stroke Filter 基础设施；
- 保留早期墨奇偏逐字/词的辅助筛选体验，不因辅码强制提交整句；
- 提供高质量中文语音输入，Voice Trigger 与 ASR Provider 解耦；
- ASR 与可选 LLM 后处理解耦；
- 数据流透明、可审计、可配置。

## 核心架构

```text
Pinyin / Shuangpin
       ↓
  LibIME Candidates
       ↓
Auxiliary Filter Trigger (`)
       ↓
Configured Auxiliary Filter
   Disabled / Stroke / MoQi
```

```text
Microphone / Long-press Space
       ↓
    Voice Trigger
       ↓
SpeechRecognizer / RecognitionService
       ↓
Configured ASR Provider
       ↓
Raw Transcript
       ↓
Optional LLM Post Processor
       ↓
IME
```

Trigger 只表达用户意图；具体 Filter / Provider 由配置决定。

## 当前阶段

Phase 1 已完成，当前处于 **Phase 2 — MoQi Auxiliary Filter V1 PoC**。

当前重点是把此前独立的 MoQi mode/trigger 重构为基于上游 Stroke Filter 的统一 Auxiliary Filter，并验证：

- Disabled / Stroke / MoQi 配置；
- MoQi selection-frontier 过滤；
- partial selection；
- composition 保留；
- Backspace / Escape；
- 继续输入并再次使用 Auxiliary Filter；
- Pinyin / Shuangpin 等价核心行为；
- Stroke 回归。

当前不修改 LibIME，不进入完整语音实现。

## 文档

- [需求规格](docs/REQUIREMENTS.md)
- [路线图](docs/ROADMAP.md)
- [技术决策](docs/DECISIONS.md)
- [研究记录](research/README.md)

## 上游与 fork

项目优先复用上游能力并缩小长期 fork 面。

- 总控仓库：`fcitx5-moqi`
- Phase 2 fork：`choicky/fcitx5-chinese-addons`
- 当前不 fork LibIME
- `fcitx5-android` 是否 fork，待后续 Voice PoC 的实际修改边界确认
