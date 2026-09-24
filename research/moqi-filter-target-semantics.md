# MoQi Filter 目标位置语义研究

## 基线

- `fcitx5-chinese-addons`: `61474bd3aa9fca26d1c31df93343035697e9f265`
- `libime`: `171edcf137001e8eb8274f53ec010058cda70b09`

## 结论

MoQi V1 **不应照搬**现有 Stroke Filter 的“候选字符串任意字符匹配”。

本项目要求的早期墨奇交互，更适合把 MoQi 定义为：

> 对当前尚未选择（unselected）的首个汉字/音节进行辅助码约束。

理由：

1. LibIME 用 `selectedLength()` 明确区分已选择前缀与剩余输入。
2. Chinese Addons 的 `buildTabActions()` 已经从 `selectedLength()` 开始解析当前首个未选音节。
3. 普通 Pinyin candidate 保存 candidate index 和 `selectLength`，可回到对应 `SentenceResult`。
4. 选择普通候选走 `selectCandidatesToCursor()`，可以形成 partial selection，随后继续处理剩余输入。
5. 因此 MoQi 无需“在整句中寻找任意匹配字”；它应约束当前 selection frontier 上的第一个输出汉字。

## V1 建议语义

假设：

```text
已选前缀 | 当前未选输入
          ^
          selection frontier
```

MoQi 输入后：

1. 只检查每个候选在 frontier 后产生的**第一个汉字**；
2. 该字的墨奇码以前缀方式匹配当前 MoQi buffer，则保留候选；
3. 后续汉字不参与本次 MoQi 判断；
4. 用户选择候选/单字后，由现有 partial-selection 状态推进 frontier；
5. 如后续仍有歧义，可再次进入 MoQi Filter。

这正好对应“逐字/词按需筛选”，并避免新版“句中任意辅码 + 整句立即上屏”的模式。

## 实现边界判断

目前证据表明，V1 所需信息均可从 `fcitx5-chinese-addons` 的候选层及现有 LibIME public context/candidate 信息获得。

因此：

- D004（复用现有辅助筛选机制）：可以从 Provisional 升为 **Accepted for V1**。
- D005（V1 尽量不修改 LibIME）：可以从 Provisional 升为 **Accepted for V1**。

这里仅针对 V1。未来若要求“指定句中任意位置辅码”才重新评估 LibIME 改动。

## 下一步

Phase 1 尚未结束。下一项按 ROADMAP 研究墨奇码表：

- 当前权威来源；
- license；
- 字符覆盖；
- 编码格式；
- 构建时转换/版本固定方式。

完成码表研究后，再形成 Phase 1 的 MoQi Filter V1 最小 patch map。
