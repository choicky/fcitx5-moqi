# Fcitx5 Pinyin Stroke Filter 源码研究

## 研究基线

上游：`fcitx/fcitx5-chinese-addons`

本次固定研究 commit：

`61474bd3aa9fca26d1c31df93343035697e9f265`

重点文件：

- `im/pinyin/pinyin.h`
- `im/pinyin/pinyin.cpp`
- `im/pinyin/pinyincandidate.h`
- `im/pinyin/pinyincandidate.cpp`
- `modules/pinyinhelper/pinyinhelper.cpp`

## 1. 已确认的整体调用链

Stroke Filter 并不进入 LibIME decoder 重新解码，而是在 Fcitx5 候选列表层做过滤。

主要链路：

```text
FilterByStroke (默认 grave / `)
    ↓
PinyinEngine::handleStrokeFilter()
    ↓
PinyinTabbedCandidateList::setStrokeFilterMode()
    ↓
PinyinEngine::updateFilter()
    ↓
CommonCandidateList::setFilter(...)
    ↓
PinyinTabbedCandidateList::filter(candidate)
    ├── filterByCheckedAction(candidate)
    └── filterByStroke(candidate)
```

输入具体笔画时：

```text
h/s/p/n/z
    ↓
handleStrokeFilter()
    ↓
映射为 1/2/3/4/5
    ↓
PinyinTabbedCandidateList::pushStroke()
    ↓
strokeBuffer_.type()
    ↓
PinyinEngine::updateFilter()
```

Android/触屏候选 tab 使用同一个核心机制：

```text
CandidateAction “笔画”
    ↓
triggerMainAction(STROKE_ACTION)
    ↓
setStrokeFilterMode()

“一/丨/ノ/㇏/𠃍”
    ↓
triggerStrokeAction()
    ↓
pushStroke()
    ↓
updateFilter()
```

这说明键盘快捷键与触屏 UI 最终共享 `PinyinTabbedCandidateList` 的 filter 状态。

## 2. Stroke Filter 的匹配语义

`filterByStroke()` 对候选文本逐 Unicode 字符遍历：

1. 对每个汉字调用 `PinyinHelper::reverseLookupStroke(chr)`；
2. 得到该字完整笔画码；
3. 判断其是否以当前 `strokeBuffer` 为前缀；
4. 候选文本中**任意一个字符**匹配，就保留整个候选。

因此当前 Stroke Filter 的真实语义不是“只过滤某个明确位置的汉字”，而是：

> 候选词/句中只要存在一个字，其笔画前缀匹配当前输入，就保留该候选。

这是 MoQi 设计必须注意的语义差异。

## 3. Stroke 数据层

Pinyin 激活时会调用：

`IPinyinHelper::loadStroke()`

`PinyinHelper::lookupStroke()` 支持两种输入：

- 数字：`1 2 3 4 5`
- 字母：`h s p n z`

映射关系：

- h -> 1
- s -> 2
- p -> 3
- n -> 4
- z -> 5

候选过滤本身主要使用反查：

`reverseLookupStroke(汉字) -> stroke code`

这对 MoQi 很有参考价值。MoQi V1 最自然的数据接口同样可以设计为：

`reverseLookupMoQi(汉字) -> moqi code`

然后进行前缀匹配。

## 4. 候选选择与 commit

这里发现了一个重要区别。

### 4.1 普通 Pinyin 候选

`PinyinCandidateWord::select()` 调用：

`context.selectCandidatesToCursor(idx_)`

然后：

`engine_->updateUI(inputContext)`

因此普通候选选择走 LibIME `PinyinContext` 的 selection 状态，并不在 CandidateWord 中直接 commit 文本。

`updateUI()` 只有在 `context.selected()` 表示整个当前输入已经完成选择时，才：

`inputContext->commitString(context.sentence())`

这与项目要求“局部选择不应天然等于整句 commit”是兼容的。

### 4.2 独立 StrokeCandidateWord

另有一套“纯笔画输入候选”逻辑。当用户输入本身全部由 h/p/s/z/n 等笔画字符组成时，可以生成 `StrokeCandidateWord`。

其 `select()` 会直接：

`commitString(hz_)`

随后 reset。

这与 **Stroke Filter** 是两套不同用途的功能。

MoQi 项目当前需要借鉴的是 **Stroke Filter**，而不是这种“纯形码直接输入汉字”的 StrokeCandidateWord 模式。

## 5. 单字过滤和拼音过滤已经存在

`PinyinTabbedCandidateList::buildTabActions()` 当前还构造：

- 当前拼音/音节相关 filter actions；
- “单字” filter；
- “笔画”入口。

`filter()` 最终把这些条件与 Stroke Filter 组合。

因此对 MoQi V1，一个很有价值的低风险路径是：

> Pinyin/Shuangpin 正常候选
> -> 可选“单字”
> -> MoQi Filter
> -> 选择普通 PinyinCandidateWord
> -> 沿用 LibIME 原有 selection/learning 行为

这样不需要让墨奇候选自己实现 commit/learning。

## 6. 对 MoQi Filter V1 的直接映射

现有 Stroke 机制可以近似映射为：

| Stroke | MoQi |
|---|---|
| `FilterByStroke` | `FilterByMoQi` |
| stroke mode | moqi mode |
| `strokeBuffer_` | `moqiBuffer_` |
| `pushStroke()` | `pushMoQi()` |
| `filterByStroke()` | `filterByMoQi()` |
| `reverseLookupStroke()` | `reverseLookupMoQi()` |
| “笔画” CandidateAction | “墨奇” CandidateAction |

`updateFilter()`、`CommonCandidateList::setFilter()`、普通 `PinyinCandidateWord::select()` 和 LibIME selection 机制原则上都可以继续复用。

## 7. 当前最重要的限制

不能简单得出“把 Stroke 表换成墨奇表就完成”的结论。

原因是现有 `filterByStroke()` 对一个多字候选采用：

> **任意字符匹配即可保留整个候选**

而项目需求更强调：

> 对当前需要处理的字/词进行墨奇筛选，并允许继续处理其他字词。

所以至少需要验证三种交互：

1. **单字候选 + MoQi**  
   最接近现有架构，风险最低。

2. **词候选 + MoQi**  
   需要确定墨奇码应约束词中的哪个字；不能未经设计直接沿用“任意字匹配”。

3. **部分选择后的下一段 + MoQi**  
   需要验证 `selectedLength()`、`candidatesToCursor()` 和当前 cursor 下的候选行为，确认能否自然实现“选好前面的字/词后继续辅码筛选后面的部分”。

第 3 项是判断能否还原用户偏好的早期墨奇交互的关键测试。

## 8. 当前结论

### 已确认

- Fcitx5 Pinyin/Shuangpin 已经具备正式的候选辅助筛选框架。
- Stroke Filter 位于 `fcitx5-chinese-addons` 候选层，而不是 LibIME decoder。
- filter 条件可组合。
- 普通 Pinyin 候选选择使用 LibIME `PinyinContext` selection，而不是 CandidateWord 直接 commit。
- 因此 **MoQi Filter V1 很有希望只修改 `fcitx5-chinese-addons`，不修改 LibIME 核心。**

### 尚未确认

- 多字词候选中 MoQi 应如何精确绑定目标字；
- partial selection 后继续 MoQi 筛选的实际行为；
- Android 端增加“墨奇” tab/action 所需的最小 UI 改动；
- 墨奇码表最终数据接口和打包方式。

## 9. 下一步

下一轮源码研究重点：

1. 追踪 `PinyinContext::selectedLength()`、`selectCandidatesToCursor()`、`candidatesToCursor()`；
2. 追踪 `ChooseCharFromPhrase` 的实现；
3. 明确 partial selection 后候选列表的生成范围；
4. 用这些结果定义“当前字/当前词”的 MoQi 约束语义；
5. 再决定 MoQi V1 是否完全不需要修改 LibIME。
