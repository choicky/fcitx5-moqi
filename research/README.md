# Research Notes

本目录用于保存源码研究和技术验证记录。

## 当前研究重点

### Fcitx5 Chinese Addons / Pinyin

优先完整追踪现有 Stroke Filter：

`FilterByStroke -> handleStrokeFilter() -> updateFilter() -> candidate filtering/selection`

重点回答：

- Stroke 数据来自哪里、如何加载；
- 输入状态如何进入/退出 filter；
- filter 如何访问并包装候选；
- 选中筛选后的候选后如何回到原候选行为；
- 是否支持 partial selection；
- Pinyin 与 Shuangpin 是否共享该机制；
- Android 端如何展示/触发 filtering；
- 将 Stroke matcher 替换/扩展为 MoQi matcher 需要修改哪些文件。

### LibIME

仅研究实现 MoQi Filter 所必需的接口，避免过早进入 decoder/lattice 内部。

### 墨奇码表

确认当前权威码表、许可证、格式、字符覆盖范围，以及适合 Fcitx5 的构建/加载方式。

### Voice

暂时作为独立研究线保留。键盘输入架构明确后，再系统研究 Fcitx5 Android 的 SpeechRecognizer/RecognitionService 路径。

## 记录原则

每份研究记录应尽量包含：

1. 上游仓库与具体 commit/tag；
2. 文件和函数位置；
3. 已确认事实；
4. 尚未确认的问题；
5. 对项目设计的影响；
6. 是否需要更新 `docs/DECISIONS.md` 或 `docs/ROADMAP.md`。

避免仅记录二手资料结论；关键技术判断尽量回到上游源码、官方文档或 issue/PR。
