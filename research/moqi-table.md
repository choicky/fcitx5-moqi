# 墨奇码表研究

## 研究基线

- 上游：`gaboolic/moqima-tables`
- main commit：`6d8ba8f1c57466f358e682baefe11bbd0fe389ab`
- commit message：`支持GB18030`
- 时间：2026-06-25

## 已确认

1. 仓库明确定位为“墨奇码的拆分码表”，README 说明已按字形递归拆分并取首末双形音托。
2. 仓库许可证为 **MIT**；LICENSE 要求再分发时保留版权和许可声明。
3. 当前仓库提供：
   - `moqima_gb18030.txt`：GB18030 范围的墨奇码数据；
   - `chaifen_gb18030.txt`：拆分数据；
   - `首尾码8105.txt`：常用集合的首尾码表；
   - `字根归并.txt`：字根到键位的归并规则。
4. `首尾码8105.txt` 格式已确认是 TSV：
   `汉字<TAB>两字母墨奇码<TAB>首形末形`。
   例如：`啊 kk 口可`、`阿 ek ⻖可`。
5. README 的键位归并覆盖 qwerty 字母键，并给出了大量字根/特殊字形映射。

## 对 V1 的数据设计

V1 运行时只需要：

`Unicode 汉字 -> 墨奇码`

不需要把完整拆分过程放进输入法运行时。

建议：

1. 构建时从固定 upstream commit 生成紧凑 lookup 数据；
2. 运行时只做单字符反查和 prefix match；
3. 保存 upstream commit、生成脚本和 MIT attribution，保证可追溯；
4. 不 fork 码表仓库，不手工维护第二份“权威码表”。

## 尚需验证

`moqima_gb18030.txt` 文件较大，当前 GitHub connector 未直接返回其内容；进入实际数据转换前必须下载/解析该文件并核对：

- 精确字段格式；
- 总字符数与重复项；
- Unicode/GB18030 覆盖；
- 是否存在一字多码；
- 异体/扩展区字符处理。

这些属于 Phase 2 前的数据验证，不影响 Phase 1 架构判断。

## Phase 1 影响

码表已有明确来源、MIT 许可和 GB18030 版本，足以继续形成 MoQi Filter V1 的最小 patch map。

下一步：汇总 Phase 1 已确认的候选筛选、partial selection、Android UI、目标位置语义和码表边界，形成最小改造设计，并逐项核对 ROADMAP Phase 1 Exit Criteria。
