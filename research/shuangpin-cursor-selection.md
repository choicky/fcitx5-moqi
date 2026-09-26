# Shuangpin cursor-boundary partial selection：定性结论

> 问题：Shuangpin 在 cursor-boundary 部分选择后继续输入新音节时，未选中的剩余音节"消失"——是 upstream 既有行为、LibIME 固有语义、还是本分支 Auxiliary Filter/MoQi 改造引入的 regression？
> 结论：**upstream/LibIME 既有语义 + 测试构造错误；不是 regression，剩余音节也没有丢失。** 不需要修改 LibIME，也不需要修改产品代码。

## 1. 复现（同一探针矩阵跑两个 ref，输出逐字相同）

探针批次：run **`36243942929`**（一次性分支 `probe/shuangpin-cursor-boundary`，workflow 同时构建并运行两个 ref；探针只打印、不断言被观察的行为）

- 腿 1：`upstream-61474bd` = `61474bd3aa9fca26d1c31df93343035697e9f265`（= merge-base）
- 腿 2：`fork-0d0102b8` = `0d0102b82b35debf4cf22ce192060c07f10e7b35`（当前分支 tip）

三组变体（都不涉及 MoQi / Auxiliary Filter）：A = Shuangpin + 光标移到音节边界后选 西；B = Shuangpin 不移动光标选 西；C = Pinyin + 显式分隔符选 西。

| 变体 | 步骤 | preedit | cursor | 候选（前若干） |
|---|---|---|---|---|
| A | `xian` | `xi an` | 5 | 西安,西岸,锡安,… |
| A | Left Left | `xi an` | **2** | 系,西,洗,细,… |
| A | 选 西 | `西an` | 3 | 安,an,按,案,… |
| A | 输入 `n` | **`西na n`** | 4 | 南,n,那,拿,… |
| A | 输入 `i` | **`西ni an`** | 5 | 你,ni,拟,泥,… |
| B | 选 西（不移动光标） | `西an` | 5 | 安,an,按,… |
| B | 输入 `n` `i` | `西an ni` | 8 | 安妮,annie,安你,**安**,按,… |
| C | 选 西（不移动光标） | `西'' an` | 8 | 安,按,案,… |
| C | 输入 `m` `e` `n` | `西'' an men` | 12 | 俺们,暗门,安门,**安**,按,… |

**两个 ref 的三组输出完全一致**（含 cursor 数值与候选列表顺序）→ 与分支改动无关。

## 2. 根因 / call chain

选择路径：

1. `im/pinyin/pinyincandidate.cpp:311` `PinyinCandidateWord::select()` → `context.selectCandidatesToCursor(idx_)`
2. libime `pinyincontext.cpp:607` `PinyinContext::selectCandidatesToCursor()` → `d->select(candidatesToCursor()[idx])`
3. libime `pinyincontext.cpp:297` `PinyinContextPrivate::select()` → `selectHelper()`（`:277`）记录 `SelectedPinyin{offset_ = selectedLength() + p->to()->index()}` → `q->update()`
4. libime `pinyincontext.cpp:868` `selectedLength()` = 最后一个选择的 `offset_`；`:672` `update()`：未全选时**只重新解析** `userInput().substr(selected_.back().back().offset_)` → 未选中部分仍会被解码（**继续输入/继续组合本来是被支持的**）
5. 光标候选来自 `:184 needCandidatesToCursor()` / `:201 updateCandidatesToCursor()`：按 `alignCursorToNextSegment()` 取到"光标对齐的下一个音节边界"为止的候选 —— 这就是光标停在边界时能选到 西 的原因

输入路径（关键）：

6. libime `pinyincontext.cpp:465` `PinyinContext::typeImpl()`：先 `auto changed = cancelTill(cursor());`
   - `:626 cancelTill(pos)` = `while (selectedLength() > pos) cancel();` → 光标在选区之前时取消选择
7. 再 `InputBuffer::typeImpl(s, length)` → **fcitx5 core** `src/lib/fcitx-utils/inputbuffer.cpp:79`：
   `input_.insert(std::next(input_.begin(), cursorByChar()), …)` → **在光标处插入，而不是追加到末尾**
8. `:510 setCursor()`（Left/Right 走的路径）= `cancelTill(pos)` + `InputBuffer::setCursor(pos)` + `candidatesToCursorNeedUpdate_ = true`

因此变体 A 的真实过程是：`xi|an`（光标 2）→ 输入 `n` → 在光标处插入 → `xi|n|an` → 重新切分 `na`+`n`（preedit `西na n`）→ 输入 `i` → `xi|ni|an`（preedit `西ni an`）。
**剩余音节 `an` 一直在 composition 里**，只是不再是首音节，所以 `安` 不在候选前列 —— 早期"音节消失"的判断源于只看候选 dump、没看 preedit/cursor。

## 3. 归类

| 候选归类 | 判定 |
|---|---|
| upstream 既有行为 | **是**（两 ref 输出逐字相同） |
| 本分支 regression | **否**（新增代码从不修改 LibIME context：`git diff 61474bd..HEAD -- im/pinyin` 中新增行对 `state->`/`context_` 只有只读使用，筛选状态放在自己的 `auxiliaryFilterBuffer_`） |
| 测试构造错误 | **是（主因）**：用 Left×2 把光标移到音节中间后再继续输入，触发的是"在光标处插入"这一**另一个交互**；Pinyin 测例把光标留在末尾，所以没有这个问题 |
| LibIME 固有语义 | 语义层面是 upstream 的"可移动 composition 光标 + 光标处插入"，属有意设计（`cancelTill(cursor())` 正是为它服务） |
| 尚无法确定 | 无 |

## 4. 对项目 Exit Criteria 的影响

- "Pinyin / Shuangpin 等价核心行为（partial selection、继续输入）"**不受影响**：变体 B 显示 Shuangpin 选择 西 后继续输入 `n`、`i` 时，剩余音节照常解码，`安` 甚至在候选列表中（`安` 为精确候选），与变体 C 的 Pinyin 行为一致。
- Phase 3 edge case 结论不变；无需新增/修改产品行为，也无需为它调整 Phase 3 的条目。
- 原"待定性"备注应替换为本结论（不再视为风险项）。

## 5. 最小调整建议（未实施，待确认）

测试侧二选一，都不改产品代码：

1. **推荐**：Shuangpin 的"继续输入"断言按变体 B 构造——选择 西 时**不移动光标**（与 Pinyin 测例同构），随后输入 `n`、`i` 并断言 `安` 相关候选存在。已实测该路径在 upstream 与 fork 上都成立。
2. 若要保留 cursor-boundary 场景，则不要再续输入新音节，或改为断言 preedit 仍含剩余音节（`西ni an`），即显式覆盖"光标处插入 + 重新切分"这一 upstream 语义。

## 6. 是否需要修改 LibIME

**不需要。** 观察到的行为是 upstream 设计的一部分（可移动光标 + 光标处插入 + 选择随光标取消），不是缺陷；本分支也没有触碰相关代码路径。
