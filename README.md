# fcitx5-moqi

面向 Android 的 Fcitx5 中文输入方案研究与实现项目。

## 目标

项目以 Fcitx5 Android 为第一目标平台，研究并实现：

- 以 Fcitx5 Pinyin/Shuangpin + LibIME 为主的中文拼音/双拼输入；
- 将墨奇码作为按需使用的辅助码，用于候选汉字筛选，而不是作为主输入编码；
- 保留早期墨奇偏逐字/词的辅码交互，避免“输入辅码即强制提交整句”；
- 高质量、可替换的语音识别（ASR）；
- ASR 与可选 LLM 后处理解耦；
- 数据流透明、可审计、可配置。

## 当前阶段

当前处于**上游源码研究与技术可行性验证阶段**。在方案明确前，不进行大规模实现。

当前优先研究：

1. `fcitx5-chinese-addons` Pinyin/Shuangpin 的 Stroke Filter 与候选筛选机制；
2. LibIME 的词典、语言模型、用户学习及候选接口；
3. 墨奇码表如何以最小侵入方式接入现有辅助筛选机制；
4. Fcitx5 Android 的语音输入与 Android SpeechRecognizer/RecognitionService 集成路径。

## 文档

- [需求规格](docs/REQUIREMENTS.md)
- [路线图](docs/ROADMAP.md)
- [技术决策](docs/DECISIONS.md)
- [研究记录](research/README.md)

## 上游项目

本项目优先复用上游能力，尽量避免维护不必要的长期 fork。是否 fork `fcitx5-chinese-addons`、`fcitx5-android` 或 LibIME，将在源码研究确认实际修改边界后决定。

> 当前仓库主要承担项目需求、研究、设计和集成工作的总控角色。
