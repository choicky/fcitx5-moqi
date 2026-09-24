# Fcitx5 Android Tabbed Candidate UI 源码研究

## 研究基线

- 上游：`fcitx5-android/fcitx5-android`
- commit：`e6199a2801b0c76e1baeff7a48f6b18e910e9dd7`
- 对照：`fcitx5-chinese-addons` commit `61474bd3aa9fca26d1c31df93343035697e9f265`

## 已确认

1. Chinese Addons 每次构造普通候选列表后，给 `CommonCandidateList` 安装
   `PinyinTabbedCandidateList`。
2. `PinyinCandidateWord::selectLength` 来自当前
   `candidatesToCursor()` 结果末节点的输入 offset，因此候选本身保留“消耗多少未选拼音”的信息。
3. `buildTabActions()` 使用 `selectedLength()` 和当前候选路径，构造当前首个未选音节相关的 filter actions，并另有“单字”“笔画”入口。
4. Android frontend 直接读取 `TabbedCandidateList::tabActions()`，把 actions 发送到 Kotlin UI。
5. Expanded Candidate UI 用 `CandidateTabActionsAdapter` 渲染这些 actions；点击后通过
   `triggerCandidateListTabAction(id)` 原样回调 Fcitx。
6. 因此 Android 已有完整通用的 TabbedCandidateList UI/回调链。新增 MoQi tab/action 原则上不需要另造一套 Android 候选筛选协议。

## 对 MoQi V1 的影响

目前最小架构进一步收敛为：

```text
LibIME Pinyin/Shuangpin candidates
  -> PinyinTabbedCandidateList
  -> 当前未选段/首音节
  -> MoQi filter state + character lookup
  -> 普通 PinyinCandidateWord selection
  -> LibIME partial selection
```

Android 侧现有 tab-action 桥接可以复用。

## 仍需确认

不能直接照搬 Stroke 的“候选任意字符匹配”。下一步只研究一个核心问题：

> 如何利用 candidate 的 selectLength / SentenceResult path，把 MoQi 约束精确绑定到当前未选段的目标字，而不破坏词候选和 partial selection。

该语义确认前不进入实现。
