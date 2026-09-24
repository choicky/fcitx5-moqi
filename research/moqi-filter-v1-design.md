# MoQi Filter V1 最小改造设计

## Phase 1 结论

基于已固定版本的上游源码研究：

- `fcitx5-chinese-addons`：`61474bd3aa9fca26d1c31df93343035697e9f265`
- `libime`：`171edcf137001e8eb8274f53ec010058cda70b09`
- `fcitx5-android`：`e6199a2801b0c76e1baeff7a48f6b18e910e9dd7`
- `moqima-tables`：`6d8ba8f1c57466f358e682baefe11bbd0fe389ab`

MoQi Filter V1 可以优先只修改 **fcitx5-chinese-addons**，不修改 LibIME 核心，也不要求修改 Android frontend 的候选协议。

## V1 交互语义

墨奇辅码约束 selection frontier 后的**第一个未选汉字**：

1. 正常使用 Pinyin/Shuangpin；
2. 需要消歧时进入 MoQi Filter；
3. 输入墨奇码前缀；
4. 只检查候选在当前未选段产生的第一个汉字；
5. 匹配则保留该候选；
6. 选择后使用现有 partial selection 推进；
7. 剩余拼音继续解码，可再次使用 MoQi。

辅码本身不等于 commit 整句。

## 最小修改范围

### 1. fcitx5-chinese-addons / Pinyin

主要修改候选/filter 层，预计集中在：

- `im/pinyin/pinyin.h`
- `im/pinyin/pinyin.cpp`
- `im/pinyin/pinyincandidate.h`
- `im/pinyin/pinyincandidate.cpp`

新增/扩展内容：

- MoQi filter mode；
- MoQi input buffer；
- `FilterByMoQi` 配置/键位；
- MoQi tab action；
- `filterByMoQi()`；
- Escape / Backspace / Return 状态处理；
- 与现有 `filterByCheckedAction()`、Stroke Filter 的组合规则。

原则：保留现有 Stroke Filter，不替换它。

### 2. MoQi lookup 数据

增加构建期数据转换：

```text
moqima-tables pinned commit
        ↓
conversion script
        ↓
compact char -> code table
        ↓
runtime reverse lookup
```

运行时不实现拆字算法。

实现位置有两个候选：

A. 扩展现有 `pinyinhelper`，增加 MoQi reverse lookup；
B. 新建独立 MoQi helper/module。

V1 优先评估 A，因为其职责与现有 stroke reverse lookup 最接近；如果会导致 pinyinhelper 职责过度耦合，再采用 B。

### 3. LibIME

**V1 不修改。**

继续使用：

- `candidatesToCursor()`
- `selectCandidatesToCursor()`
- `selectedLength()`
- `selectCustom()`
- `cancel()`

Pinyin/Shuangpin 共用现有 partial-selection 状态机。

### 4. Fcitx5 Android

**V1 原则上不修改 candidate frontend 协议。**

复用现有：

`TabbedCandidateList -> CandidateAction -> Android Expanded Candidate UI -> triggerCandidateListTabAction()`

若 PoC 发现仅布局/可用性需要调整，再作为 Phase 3 Android UX 工作处理，不提前纳入 V1 核心 patch。

## 必须避免

V1 不做：

- 句中任意位置辅码；
- 输入辅码后自动 commit 整句；
- 修改 LibIME decoder/lattice；
- 自建 Android 候选筛选协议；
- 把墨奇变成主输入编码；
- 同时重构词库/语言模型；
- 语音输入代码。

## 初始测试矩阵

至少覆盖：

1. 全拼 + 单字 MoQi 筛选；
2. 双拼 + 单字 MoQi 筛选；
3. 多字候选只按 frontier 首字过滤；
4. 选择后 composition 未被整句 commit；
5. selection 后继续输入；
6. selection 后再次 MoQi；
7. Backspace 删除 MoQi code；
8. Escape 退出 MoQi mode；
9. Stroke Filter 仍正常；
10. 普通 Pinyin/Shuangpin 无 MoQi 时行为不变；
11. 用户词频学习不受破坏；
12. Android tab action 可触发完整 MoQi filter 流程。

## Phase 1 Exit Criteria

### 1. 修改哪些上游组件？

已回答：V1 主要修改 `fcitx5-chinese-addons`；使用 `moqima-tables` 数据。

### 2. 是否需要 fork？

进入 Phase 2 时建议 fork `fcitx5-chinese-addons`，建立独立 feature branch。暂不 fork LibIME 和 fcitx5-android。

### 3. 是否需要修改 LibIME？

V1 不需要。已有 public context/candidate/partial-selection 能力满足当前设计。

### 4. 码表如何加载？

固定 `moqima-tables` commit，构建时转换为紧凑的 char -> code lookup 数据；具体二进制格式在 PoC 前通过真实文件解析确定。

### 5. Android 如何暴露/配置？

复用现有 TabbedCandidateList / CandidateAction 通道。MoQi 作为与“笔画”同级的过滤 action；物理键盘键位另由 `FilterByMoQi` 配置。

### 6. 第一版需要哪些测试？

已列出 12 项初始测试矩阵。

## Phase 1 状态

**Exit Criteria 已满足，可以结束 Phase 1。**

下一阶段是 Phase 2 — MoQi Filter PoC。

Phase 2 开始前第一步不是写大规模代码，而是：

1. fork `fcitx5-chinese-addons`；
2. 建立 `feature/moqi-filter`；
3. 实际解析 `moqima_gb18030.txt`，确认格式/重复项/一字多码；
4. 再实施最小 PoC。
