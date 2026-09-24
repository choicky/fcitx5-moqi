# LibIME Pinyin partial selection 源码研究

## 研究基线

- 上游：`fcitx/libime`
- master commit：`171edcf137001e8eb8274f53ec010058cda70b09`（2026-09-24）
- 重点：`src/libime/pinyin/pinyincontext.cpp/.h`

## 已确认

1. `candidatesToCursor()` 并非简单返回整句候选。cursor 位于未完成输入中间时，LibIME 会按当前 cursor 截断/构造 partial candidates。
2. `selectCandidatesToCursor(idx)` 直接选择上述 partial candidate。
3. selection 被记录在 `selected_`；`selectedLength()` 成为后续解码起点。
4. `update()` 在已有 selection 时，从最后已选 offset 后的剩余输入重新建立 Pinyin/Shuangpin SegmentGraph 并 decode。
5. 因此“先选前一段，再继续处理后一段”是 LibIME 原生状态机的一部分，并非需要 MoQi 自己模拟。
6. `selectCustom(inputLength, segment, encodedPinyin)` 同样生成一个 selection segment；`cancel()` 可撤销最后一次 selection。
7. Pinyin 与 Shuangpin 在这里共享同一个 `PinyinContext`。差别主要在 `update()` 解析剩余输入时选择 `parseUserPinyin()` 或 `parseUserShuangpin()`。

## 对 MoQi 的阶段性结论

这进一步支持 MoQi V1 保持在 `fcitx5-chinese-addons` candidate/filter 层：

```text
Pinyin/Shuangpin decode
  -> 当前/到 cursor 的候选
  -> MoQi 过滤
  -> 选择 partial candidate
  -> LibIME 保留 selection
  -> 对剩余输入继续 decode / 筛选
```

因此目前仍未发现必须修改 LibIME 核心的理由。

## 尚未解决

关键剩余问题是 MoQi filter 的“目标位置”语义。现有 Stroke Filter 是候选字符串任意字符匹配；MoQi 不能未经设计直接照搬。

下一步按 Phase 1 研究：

1. fcitx5-chinese-addons 如何构造 `PinyinTabbedCandidateList` 及 candidate selectLength；
2. Android 如何渲染/触发 tab actions；
3. 在上述证据基础上定义 MoQi 的“当前待选字/词”过滤规则。

在这些完成前，不进入实现。
