# fcitx5-moqi 进度汇总（供 ChatGPT 阅读与管理）

> 生成时间：2026-09-25（覆盖上一版快照）。所有事实已用仓库/CI/发布页核验，附 commit 与 run 编号。
> 权威进度文档：仓库内 `docs/ROADMAP.md`；本文件是给外部协作者的汇总。

## 0. 一句话状态

面向 Android 的 Fcitx5 中文输入方案（Pinyin/Shuangpin 主输入 + 墨奇码按需辅助筛选 + 后续语音）。**Phase 0/1/2 已完成，Phase 3 进行中，只剩一项需设备的实机回归**；已有可安装的正式签名发布包。

## 1. 坐标（已核验）

| 项 | 值 |
|---|---|
| 总控仓库 | `choicky/fcitx5-moqi` → `main@3040944`（干净、已推送） |
| addon fork | `choicky/fcitx5-chinese-addons` → `feature/moqi-filter@0d0102b82b35debf4cf22ce192060c07f10e7b35`（干净、已推送；PR #1 OPEN/Draft，4 项检查全 SUCCESS） |
| 相对 upstream | 基线 `61474bd`（= merge-base，落后 0 / 领先 37+），净 diff **15 文件 +885/−102** |
| Android fork | `choicky/fcitx5-android` → `moqi-test-apk@59efbf543d1ca47041886794e085cef703bde180` |
| 发布 | `v0.1.3-moqi.2`（Latest，正式签名，包名 `.moqi`）、`v0.1.3-moqi.1`、`v0.1.3-moqi-test.1`（pre-release，debug） |
| 签名密钥 | 本机 `...\default-workspace\signing\`（PKCS#12 + 口令），证书指纹 `C9:01:22:D6:…:AD:A7`；GitHub secrets `SIGN_KEY_BASE64/PWD/ALIAS` 已配 |

## 2. 已完成的工作（附证据）

**A. 解开 Phase 2 的 CI 阻塞（根因是测试设计问题，非产品缺陷）**
`testpinyin.cpp:320` 要求「per-IM 配置在 `reloadConfig()` 后仍保留」，但测试框架以 `SkipUserPath` 构造 `StandardPaths`，没有可写的用户配置路径 → 写不进读不到 → `load()` 把选项 reset 为默认值；分支未改持久化路径。且 `FCITX_ASSERT` 会中止整个测试二进制，该测试排在 `main()` 第 5 位 → **其后 8 个测试（含全部 Stroke/MoQi）从未执行**（这正是 ROADMAP 里 Stroke 回归长期勾不上的真因）。
修复：`f91853e`（删失效断言 + 注释）、`f903176`。证据：run `36119436393`、`36121191299` 三 job 全绿、ctest 9/9。

**B. Phase 2 收尾**：ROADMAP/DECISIONS 更新（D023）、PR 描述改写；**用户实机验证通过**（三选一显示 / 重启后保持 / 按反引号触发后按墨奇码筛选 / Stroke·Disabled 回归）→ Phase 2 = COMPLETE。

**C. Phase 3：Android 码表分发（三次尝试，最终 Android 零改动）**

| 尝试 | 结果 |
|---|---|
| ① addon 加 `COMPONENT config`（保持构建期下载） | ❌ run `36130867729`：`installLibraryConfig` 只先构建 `generate-desktop-file` 就 `--install --component config`，早于 addon 原生构建 → 文件尚不存在；已 revert `619c7b4` |
| ② Android app 侧 configure 取表 + `prebuilt-assets` | ✅ run `36131495456`，但给 Android 侧引入 MoQi 专属改动 → 弃用 |
| ③ **addon configure 阶段取表 + `COMPONENT config`** | ✅ run `36135660468`：**`app/src/main/cpp/CMakeLists.txt` 与上游逐字一致** |

pin 单一来源 `modules/pinyinhelper/moqima-gb18030.cmake`；本机以 cmake 3.31.6 实跑该段原文（下载 1,436,812 字节、哈希一致、重复执行不重下）。**结论：码表分发不再需要 fork `fcitx5-android`。**

**D. Phase 3：edge cases 补测**：5 组测试（触发入口守卫 / 两位码上限 / 无匹配即过滤 / 筛选模式下修饰键被吞 / 翻页进出筛选），断言走 `InputPanel::auxUp()` 黑盒。run `36143794062` 三 job 全绿、ctest 9/9。
审计结论：① 固定码表无重复字符 → 「一字多码」不存在（`moqi.cpp` 的 `onlyMatch` 不可达）；② 候选列表每次输入事件重建 → 无长期陈旧 tab 缓存；③ `filterByMoQi` 除 `StrokeCandidateWord` 外无豁免。

**E. Phase 3：自构建发布线（D025）**：release 包名固定 `.moqi`、debug `.debug`（提交在 fork 源码，不再由 CI 打补丁）；固定签名密钥经上游 `SIGN_KEY_*` 接口消费（fork 内无签名代码）；tag 触发 workflow → 构建 → 校验（签名 / 包名 / 码表哈希）→ 建 Release 附 APK。对照实测：CI debug 构建每次密钥都不同（3 个 APK 的 `CERT.RSA` 哈希互异）。

**F. 收口审计 + Batch A/B**

- **移除失效的 `.gitignore` 条目** → 相对 upstream 净 diff 由 16 文件 886 行降为 **15 文件 885 行**；最终提交 `0d0102b8…`，完整 PR CI run `36158212213` 三 job 全绿。
- **发布复现性修复**：`release-apk.yml` 固定 addon commit（`env.ADDON_COMMIT`，完整 SHA）+ 步骤内断言检出结果 + 验证脚本改为**从 addon pin 文件读取码表哈希**（消除重复事实来源）+ Release notes 自动写入三项来源。验证：tag `v0.1.3-moqi.2` → run `36158362313` 全通过（日志 `addon commit checked out: 0d0102b8…`；`pinned table sha256` == 包内值）；**`v0.1.3-moqi.1` 未被改动**。
- 评估文档：`research/upstream-fork-assessment.md`（四类改动归属 + 四个特定问题答复 + 复现性审计 + 快速 CI 回路评估）。

## 3. 待完成

| # | 事项 | 归属 | 说明 |
|---|---|---|---|
| 1 | **Android 实机回归（v0.1.3-moqi.2）** | 需用户设备 | 步骤与记录表：`docs/phase3-device-verification.md`；通过后 Phase 3 可置 COMPLETE |
| 2 | Auxiliary Filter 配置体验 / 性能稳定性 | 需设备 | Phase 3 范围项 |
| 3 | 定性 Shuangpin 问题（见第 4 节） | 需决策 | 定性后再补测试断言 |
| 4 | 上游贡献落地（可选） | 可案头 | 三项：Trigger/实现解耦（先只含 Disabled/Stroke）、config 契约测试、`ctest --output-on-failure` |
| 5 | Phase 4 语音 PoC 及之后 | 未开始 | 当前明确不启动 |
| 6 | 快速 CI 回路 | 暂缓 | 按指示暂缓；方案已评估（≈18min → 6–7min 带 ccache） |

## 4. 一个待定性问题（未写成断言）

Shuangpin 在 **cursor-boundary 部分选择**之后继续输入新音节时，**未选中的剩余音节从 composition 中消失**（同期无 `Commit:` 记录），候选列表只剩新音节的候选；而 Pinyin + 显式分隔符路径下同样步骤保留该音节。
证据：run `36155704509` 的断言 dump（`candidates: 你,ni,拟,泥,…`）。原计划的 Shuangpin「继续输入」断言已回退（`0d0102b8…`）——不把未定性的行为写进测试。
**待决策：预期行为还是缺陷。**

## 5. 关键决策（`docs/DECISIONS.md`）

D004 统一 Auxiliary Filter；D005 Trigger 与 Filter 解耦；D006 selection-frontier 语义；D008 码表固定来源；D010 复用 Android generic config；**D023** 持久化覆盖边界（单元测试不覆盖磁盘持久化，由实机覆盖）；**D024** 码表分发机制；**D025** 发布线（独立包名 + 自有固定密钥 + tag 触发）。

## 6. 避免重复踩坑

- **治理**：ROADMAP 管进度/下一步；DECISIONS 只记已接受决策；完成状态必须有可核查证据；不得把未运行/跳过的检查写成通过；不靠反复推 CI 猜着修；不强推。
- **环境**：开发机无 Linux 构建链（无 cmake/clang/g++/WSL/docker）→ addon 编译/测试只能用 GitHub Actions；汇报须区分「CI 实测 / 源码核对 / 用户实机 / 未验证」。git 走 https 需 `-c http.sslBackend=openssl`；`gh` 偶发网络抖动，重试即可。
- **死路（勿重试）**：用 `COMPONENT config` 分发**构建期生成**的文件；认为预测/云端候选在 MoQi 下有豁免。
- **非缺陷行为**：触发键在无候选列表或 `Disabled` 时按字面输入反引号。

## 7. 建议下一步

1. 用户用 `v0.1.3-moqi.2` 做实机回归（记录表见 `docs/phase3-device-verification.md`）→ 勾掉 Phase 3 最后一项。
2. 定性第 4 节的 Shuangpin 问题（预期行为 or 缺陷）→ 相应补测或修复。
3. 之后再议 Phase 4（语音）与上游贡献落地；快速 CI workflow 待明确指示。
