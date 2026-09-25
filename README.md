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

Phase 2 已完成（Exit Criteria 全部满足，含 Android 实机验证），当前处于 **Phase 3 — Full MoQi / Android Integration**。

当前重点：

- Android 侧墨奇表的正式分发：正常构建（不依赖 CI 临时步骤）产出的 APK 必须内含 `moqima_gb18030.txt`；
- 固定 MoQi table 的 build-time transformation 与版本更新策略；
- Android 构建、安装与实际输入体验，含 Auxiliary Filter 配置体验；
- edge cases 补测，以及向上游贡献 / 长期 fork 必要性评估。

当前仍不修改 LibIME，不进入完整语音实现。

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
