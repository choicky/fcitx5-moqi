# 墨奇码表研究

## 研究基线

- 上游：`gaboolic/moqima-tables`
- main commit：`6d8ba8f1c57466f358e682baefe11bbd0fe389ab`
- commit message：`支持GB18030`
- 时间：2026-06-25
- 实测文件：`moqima_gb18030.txt`
- SHA-256：`66deab4aaba1285e3c85eb3a364c21bc08db1911b61df8e934f0d006ca7e7923`

## 已确认

1. 仓库明确定位为“墨奇码的拆分码表”，README 说明已按字形递归拆分并取首末双形音托。
2. 仓库许可证为 **MIT**；LICENSE 要求再分发时保留版权和许可声明。
3. 当前仓库提供：
   - `moqima_gb18030.txt`：GB18030 范围的墨奇码数据；
   - `chaifen_gb18030.txt`：拆分数据；
   - `首尾码8105.txt`：常用集合的首尾码表；
   - `字根归并.txt`：字根到键位的归并规则。
4. `moqima_gb18030.txt` 已对固定 commit 的实际文件完成完整解析：
   - UTF-8；
   - 96,351 行；
   - 每行严格 3 列 TSV：`汉字<TAB>墨奇码<TAB>拆分信息`；
   - 96,351 个唯一字符，无重复字符；
   - 每个字符恰好一个墨奇码；
   - 墨奇码全部为 2 个 ASCII 小写字母；
   - 676 种二字母组合全部出现；
   - 空字段、非法码、列数异常均为 0；
   - 字符码点范围从 U+3007 到 U+323AF（范围值不表示中间每个码点均存在）。
5. `首尾码8105.txt` 同样采用 TSV 结构，例如：`啊 kk 口可`、`阿 ek ⻖可`。
6. README 的键位归并覆盖 qwerty 字母键，并给出了大量字根/特殊字形映射。

## 对 V1 的数据设计

V1 运行时数据模型可以固定为：

`Unicode code point -> 2-byte MoQi code`

因此不需要处理一字多码、变长码，也不需要把完整拆分算法放入输入法运行时。

建议：

1. 构建时从固定 upstream commit 生成紧凑 lookup 数据；
2. 运行时只做单字符反查和 1/2 字母 prefix match；
3. 保存 upstream commit、源文件 SHA-256、生成脚本和 MIT attribution，保证可追溯；
4. 不 fork 码表仓库，不手工维护第二份“权威码表”。

## Phase 2 结论

码表数据验证已完成。可以开始 MoQi Filter V1 最小 PoC：

1. 在 `choicky/fcitx5-chinese-addons` 的 `feature/moqi-filter` 分支实现；
2. 先建立可复现的码表转换/lookup；
3. 再接入 Pinyin/Shuangpin candidate-list filter；
4. V1 不修改 LibIME 核心，不修改 Android candidate frontend protocol。
