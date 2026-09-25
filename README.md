# fcitx5-moqi

面向 Android 的 Fcitx5 中文输入方案研究与实现项目。

## 目标

项目以 Fcitx5 Android 为第一目标平台，研究并实现：

- 以 Fcitx5 Pinyin/Shuangpin + LibIME 为主的中文拼音/双拼输入；
- 将墨奇码作为按需使用的辅助码，用于候选汉字筛选，而不是作为主输入编码；
- 保留早期墨奇偏逐字/词的辅码交互，避免“输入辅码即强制提交整句”；
- 高质量、可插拔 Provider 的语音识别（ASR），由用户在 Android UI 中选择；
- ASR 与可选 LLM 后处理解耦；
- 数据流透明、可审计、可配置。

## 当前阶段

Phase 1 上游源码研究已完成，当前处于 **Phase 2 — MoQi Filter V1 PoC**。

当前批次优先完成：

1. MoQi 候选过滤与状态切换；
2. Backspace/退出辅码状态；
3. 与现有 Stroke Filter 共存；
4. Pinyin/Shuangpin 关键行为测试；
5. 验证过滤后可继续输入、再次使用辅码且不强制整句 commit。

当前暂不进入 Android UI 或语音实现。

## 文档

- [需求规格](docs/REQUIREMENTS.md)
- [路线图](docs/ROADMAP.md)
- [技术决策](docs/DECISIONS.md)
- [研究记录](research/README.md)

## 上游项目

本项目优先复用上游能力，尽量避免维护不必要的长期 fork。Phase 1 已确认并已 fork `fcitx5-chinese-addons` 用于 MoQi Filter V1 PoC；当前不 fork `fcitx5-android` 或 LibIME。

> 当前仓库主要承担项目需求、研究、设计和集成工作的总控角色。
