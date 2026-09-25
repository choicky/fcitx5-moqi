# Roadmap

> 本路线图描述当前研究顺序，不代表所有阶段都必须按原方案实施。源码研究结果可以触发路线调整。

## Phase 0 — 需求确认

**状态：基本完成**

- [x] Android 作为第一目标平台
- [x] 拼音/双拼作为主输入方式
- [x] 墨奇作为按需辅助码
- [x] 明确偏好早期逐字/词辅码交互
- [x] 明确辅码不应强制 commit 整句
- [x] 明确语音可联网，但数据流必须透明可控
- [x] 明确 ASR 与 LLM 后处理解耦

## Phase 1 — 上游源码研究

**状态：IN PROGRESS**

> 当前重点：完成 Stroke Filter → CandidateList/composition 调用链与 partial selection 研究，满足退出条件后再进入 Phase 2。

### 1.1 Fcitx5 Chinese Addons

重点追踪：

- [ ] Pinyin/Shuangpin 输入链路
- [x] Stroke Filter 的完整实现
- [x] `FilterByStroke`
- [x] `handleStrokeFilter()`
- [x] `updateFilter()`
- [x] 候选过滤/包装/选择机制
- [ ] partial selection / 从词候选选字相关行为
- [ ] Android 构建中相关功能是否完整可用

目标：确定 MoQi Filter 是否可以主要在 `fcitx5-chinese-addons` 层完成。

### 1.2 LibIME

- [ ] 确认 PinyinContext 与候选接口
- [ ] 确认词典、Language Model、用户学习机制
- [ ] 确认 MoQi Filter V1 是否无需修改 LibIME
- [ ] 只有确有必要时才研究 decoder/lattice 约束接口

### 1.3 墨奇码表

- [ ] 确认权威/当前码表来源
- [ ] 核实许可证与再分发条件
- [ ] 明确“汉字 -> 墨奇码”的数据结构
- [ ] 设计构建时转换和版本固定方式

### Phase 1 退出条件

形成一份明确的 MoQi Filter 最小改造设计，回答：

1. 修改哪些上游组件；
2. 是否需要 fork；
3. 是否需要修改 LibIME；
4. 码表如何加载；
5. Android 如何暴露/配置该功能；
6. 第一版需要哪些测试。

## Phase 2 — MoQi Filter PoC

**状态：未开始**

目标：

- [ ] Pinyin 可使用 MoQi Filter
- [ ] Shuangpin 可使用 MoQi Filter
- [ ] 墨奇码只负责候选筛选
- [ ] 使用辅码不会强制提交整句
- [ ] 不破坏原有 Stroke Filter
- [ ] 不影响 LibIME 原有词库、LM 和用户学习
- [ ] Backspace 可撤销辅码/过滤状态
- [ ] 辅码过滤后可继续输入 Pinyin/Shuangpin
- [ ] 可继续对后续其他字/词再次使用辅码
- [ ] 先用最小墨奇码表验证状态机，再接入完整码表

PoC 完成后，根据修改边界决定是否 fork `fcitx5-chinese-addons` 并维护 feature branch。

## Phase 3 — Android 集成与输入体验验证

**状态：未开始**

- [ ] Android 构建与安装
- [ ] 候选栏/过滤入口交互
- [ ] 连续输入
- [ ] 词语与单字辅助筛选
- [ ] Backspace/取消选择
- [ ] 用户词频学习
- [ ] 性能与稳定性
- [ ] 与早期墨奇实际使用体验比较

只有实际体验证明必要时，再研究更复杂的 composition 内定位和局部编辑。

## Phase 4 — Voice Architecture

**状态：未开始**

架构固定为 `Audio Capture -> ASR Provider -> Raw Transcript -> Optional LLM Post Processor -> IME`。ASR Provider 从一开始可插拔，不绑定单一厂商或模型，最终由用户在 Fcitx5 Android UI 中选择。

- [ ] 跟踪 Fcitx5 Android 当前语音输入上游实现
- [ ] 研究 Android SpeechRecognizer / RecognitionService
- [ ] 定义独立 ASR Provider 接口
- [ ] 定义 Fcitx5 Android UI 的 Provider 选择与配置入口
- [ ] 定义录音、上传、停止和审计边界
- [ ] 确定是否可避免修改 Fcitx5 Android 主程序

## Phase 5 — ASR PoC

**状态：未开始**

选取有代表性的本地/云端方案验证：

- [ ] 豆包 / 阿里云 / 腾讯云 / 讯飞等代表性云端 ASR
- [ ] OpenAI-compatible ASR
- [ ] sherpa-onnx / FunASR / SenseVoice 等本地或自建 ASR
- [ ] provider 切换
- [ ] 流式/非流式识别
- [ ] 中文识别质量与延迟
- [ ] 数据流可见性

## Phase 6 — Optional LLM Post-processing

**状态：未开始**

- [ ] 与 ASR 完全解耦
- [ ] 可关闭
- [ ] 纠错/标点/格式化
- [ ] 可配置 provider
- [ ] 明确发送给 LLM 的文本范围和隐私边界

## 工程节奏

- 相关改动尽量组成逻辑完整的批次后再 push/触发 CI。
- 本地可完成的检查优先本地执行；纯文档改动原则上不触发耗时构建。
- CI 作为阶段性验证节点使用；可并行的验证尽量一次触发，避免每个小改动都等待 Actions。
- 批量不等于堆积不可审查的大改动，每个批次仍需目标明确。

## 当前下一步

继续追踪 `PinyinContext::selectedLength()`、`selectCandidatesToCursor()`、`candidatesToCursor()` 以及 `ChooseCharFromPhrase`，验证 partial selection 后继续使用 MoQi Filter 的可行性，并据此定义 **MoQi Filter V1 的目标字/词约束语义**。
