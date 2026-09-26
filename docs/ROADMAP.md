# Roadmap

> 路线图按当前已验证架构安排；源码研究或 PoC 结果可以触发有记录的调整。

## Phase 0 — 需求确认

**状态：COMPLETE**

已明确 Android first、Pinyin/Shuangpin 主输入、MoQi Auxiliary Filter、早期逐字/词交互、Trigger/Implementation 解耦、Voice Trigger、ASR/LLM 解耦及隐私边界。

## Phase 1 — 上游源码与架构研究

**状态：COMPLETE**

已完成 Phase 2 所需关键研究：

- Stroke Filter → CandidateList → composition 调用链；
- `CommonCandidateList::setFilter()`；
- LibIME partial selection / `selectedLength()` / `candidatesToCursor()` / `selectCandidatesToCursor()`；
- Android TabbedCandidateList plumbing；
- Android generic Fcitx config UI；
- MoQi target semantics；
- MoQi table 来源、版本、许可证和实际数据验证；
- MoQi reverse lookup foundation；
- Fcitx5 Android 现有 voice UI 与 upstream SpeechRecognizer 工作；
- Android SpeechRecognizer / RecognitionService 边界。

结论：Phase 2 优先只修改 `fcitx5-chinese-addons`，当前无需修改 LibIME 或 Android candidate protocol。

## Phase 2 — MoQi Auxiliary Filter V1 PoC

**状态：COMPLETE**

### 目标

把此前平行的 MoQi/Stroke mode 设计重构为基于上游 Stroke Filter 的统一 Auxiliary Filter，并完成真实端到端 PoC。

### 当前批次

- [x] 将现有 Stroke trigger/mode 最小泛化为 Auxiliary Filter；
- [x] 增加 Disabled / Stroke / MoQi 配置；
- [x] 保留 Stroke-specific filter；
- [x] 接入现有 MoQi reverse lookup；
- [x] 实现 selection-frontier MoQi filtering；
- [x] Backspace / Escape；
- [x] partial selection；
- [x] composition 保留；
- [x] 继续输入；
- [x] 再次使用 Auxiliary Filter；
- [x] Pinyin / Shuangpin 等价核心行为；
- [x] Stroke regression；
- [x] 验证 generic Android config exposure（进程内契约 + Android 实机）。

实现已提交到 `choicky/fcitx5-chinese-addons` 的 `feature/moqi-filter` 分支，当前 tip 为 `f903176f8ffe970bd9e4baf3d974d6b3828c85c5`，对应 PR #1。源码已包含上述已勾选能力及 Pinyin/Shuangpin 自动化测试。

CI（run `36121191299`，tip `f903176`）三个 job 全部通过：clang-format、Build and test (gcc)、Build and test (clang)；ctest 9/9 全部通过，其中 `testpinyinhelper` 用固定码表验证墨奇反查，`testpinyin` 覆盖 Stroke / MoQi / Disabled、partial selection、继续输入与 config 契约。

Stroke regression 的证据是上游既有测试 `testActionInStrokeFilter`、`testPinyinTabFilter`、`testPinyinTabFilterWithSeparator` 在重构后通过（仅新增显式设置 `AuxiliaryFilter=Stroke`，其余过滤流程未改）。

此前 `testAuxiliaryFilterConfigContract` 含一条在该测试环境下结构性无法成立的断言（`reloadConfig()` 之后按输入法配置取值），触发 `FCITX_ASSERT` 中止整个测试二进制，导致其后所有 Stroke / MoQi 测试从未执行。该断言已移除并在测试源码中注明原因。

Android generic config exposure 目前只有进程内验证：config descriptor 把 AuxiliaryFilter 暴露为 Enum（Type / DefaultValue / Enum[i] / EnumI18n[i]），且 `setConfigForInputMethod()` → `getConfigForInputMethod()` 对 Disabled / Stroke / MoQi 三个值往返一致。磁盘持久化与 reload 后的取值无法在单元测试框架内验证：测试环境以 `SkipUserPath` 构造 `StandardPaths`，`userPath(PkgConfig)` 为空，`safeSaveAsIni()` 无处可写、`readAsIni()` 读不到文件，`Configuration::load()` 于是把所有选项 reset 为默认值。该项已于 2026-09-25 在 Android 实机（arm64-v8a 测试 APK）验证通过：`Auxiliary Filter` 显示为 Disabled / Stroke / MoQi 三选一，重启后保持所选值，按反引号触发后按墨奇码正常筛选候选，Stroke 与 Disabled 回归正常。该结果由项目所有者实机验证并报告，非自动化测试得出。

为实机验证准备了测试 APK：`choicky/fcitx5-android` 分支 `moqi-test-apk` 的 workflow `MoQi test APK` 会把 `fcitx5-chinese-addons` submodule 切到 `feature/moqi-filter`、把墨奇表按固定 commit 与 SHA256 放进 prebuilt assets、给 debug 包加 `.debug` 包名后缀（可与官方应用共存），产出 arm64-v8a debug APK，并在打包后断言 APK 内含码表、auxiliary filter 代码与 `.debug` 包名。最近一次成功运行：run `36127155083`，对应 addon 提交 `f903176`。

Android 侧无需改动，已在源码层确认：`ConfigType.kt` 的 `"Enum" -> TyEnum`、`ConfigDescriptor.kt` 读取 `Enum` / `EnumI18n`、`PreferenceScreenFactory.kt` 把 `ConfigEnum` 渲染为 `ListPreference`，与本分支 descriptor 输出一致。

打包机制上有一个需要后续处理的发现：`fcitx5-android` 只安装 CMake 的 `prebuilt-assets` / `config` / `translation` 三个 component，addon 中不带 COMPONENT 的 `install(FILES ...)` 不会进入 APK，因此墨奇表在 Android 上必须经由 `fcitx5-android/prebuilt` 的 `chinese-addons-data`（或等效机制）分发。测试 APK 目前用 CI 临时步骤绕过，正式集成方案见 Phase 3。

### 预计修改边界

主要：`fcitx5-chinese-addons`

当前不修改：

- LibIME；
- Android candidate frontend protocol；
- MoQi-specific Android UI。

### Exit Criteria

以下流程端到端成立：

```text
Pinyin/Shuangpin
→ candidates
→ Auxiliary Filter Trigger
→ configured MoQi
→ real MoQi code
→ candidate filtering
→ partial selection
→ composition preserved
→ continue input
→ Auxiliary Filter Trigger again
→ second filtering/selection
```

同时要求 Disabled、Stroke、Backspace、Escape 均正确，测试只使用固定真实 MoQi table 数据。

Phase 2 Exit Criteria 满足前，不进入完整 Android UI 或 Voice 实现。

## Phase 3 — Full MoQi / Android Integration

**状态：COMPLETE**

### 目标

让墨奇表与 Auxiliary Filter 在 Android 上通过**正常构建**（不依赖任何 CI 临时步骤）即可用，并完成 Android 实机回归与后续集成决策。

### 当前批次

- [x] Android 侧墨奇表分发：addon 在 **configure 阶段**按固定上游 commit 取表并校验 SHA256，再以 `config` component 安装；`fcitx5-android` 本就会把 addon 的该 component 装进 APK assets，因此 **Android 侧无需任何代码改动**（`app/src/main/cpp/CMakeLists.txt` 与上游逐字一致）。正常构建的 APK 由 CI run `36135660468` 断言：`assets/usr/share/fcitx5/pinyinhelper/moqima_gb18030.txt` 存在且 SHA256 = `66deab4aaba1285e3c85eb3a364c21bc08db1911b61df8e934f0d006ca7e7923`；
- [x] 版本更新策略：pin（commit / SHA256 / URL）收敛到 `modules/pinyinhelper/moqima-gb18030.cmake` 单一文件，更新流程写在文件注释中。本地以 cmake 3.31.6 实际执行该段配置逻辑验证（下载 1,436,812 字节、哈希与 pin 一致；重复执行不重新下载，mtime 未变）；addon CI run `36135635931` 三个 job 全绿（Linux 构建 + 9/9 测试，含从安装树加载码表的 `testpinyinhelper`）；
- [x] Android 实机回归：新分发机制下配置项显示、持久化与过滤行为不变。**项目所有者人工回报（2026-09-25）：12 项操作 12/12 全部通过**，所用包为正式发布 `v0.1.3-moqi.2`；**证据为人工回报，未附截图或 logcat，非自动化测试结果，亦非本仓库维护者复测**。操作与记录表见 `docs/phase3-device-verification.md`；
- [x] 自构建发布线：固定 `.moqi` 包名 + 自有稳定签名密钥（仓库 secrets，复用上游 `SIGN_KEY_*` 接口）+ tag 触发发布 workflow。首个正式发布 `v0.1.3-moqi.1`（[release](https://github.com/choicky/fcitx5-android/releases/tag/v0.1.3-moqi.1)）：CI 校验签名存在、`.moqi` 包名、码表 SHA256 = pin；本机另行确认 APK 内签名证书指纹与自有密钥一致（`C9:01:22:D6:52:B6:24:FA:E7:04:0A:78:69:EF:C1:D6:CD:15:06:D2:80:3D:58:B1:9F:F4:59:9D:2A:BB:AD:A7`）；
- [x] edge cases 补测：新增 5 组测试（触发入口守卫、MoQi 两位码上限、无匹配即过滤、筛选模式下修饰键被吞、翻页进出筛选）；断言统一走核心 `InputPanel::auxUp()`（用户可见的辅助栏），不依赖引擎内部。CI run `36143794062` 三个 job 全绿、ctest 9/9。
- [x] 发布复现性：`release-apk.yml` 固定 addon commit（**完整 SHA**，`env.ADDON_COMMIT`），并在 workflow 内断言检出结果；Release notes 自动写入 addon commit / 码表 SHA256 / Android commit。证据：tag `v0.1.3-moqi.2` → run `36158362313`（日志 `addon commit checked out: 0d0102b82b35debf4cf22ce192060c07f10e7b35`，`pinned table sha256` 与包内一致）；`v0.1.3-moqi.1` 的 tag/APK 未被改动；
- [x] 收口审计清理：删除 `.gitignore` 中已失效的墨奇表条目（新的 configure-time fetch 不再写源码目录）→ 相对 upstream 的净 diff 由 16 文件降为 **15 文件 +885/−102**；addon 最终提交 `0d0102b82b35debf4cf22ce192060c07f10e7b35` 的完整 PR CI run `36158212213` 三个 job 全绿；
- [x] 双拼"部分选择后继续输入"断言（按 `research/shuangpin-cursor-selection.md` 的**变体 B**：选择 **西** 前不移动光标，随后输入 `n`、`i`，断言 composition 保留且候选仍含 **安**；不涉及辅助筛选，也不把 cursor-boundary 插入场景写成末尾追加）：commit `8ccb09f01da675eda53527136d07a977eb63320e`（仅 `test/testpinyin.cpp`，+39 行），完整 PR CI run `36246062225` 三个 job 全绿、ctest 9/9（`testpinyin Passed 4.03 sec`）；

> edge case 审计结论：① 固定码表无重复字符，因此「一字多码」边界在 V1 不存在，`moqi.cpp` 的 `onlyMatch` 分支用真实码表不可达（防御性代码）；② 候选列表每次输入事件都会重建，不存在长期陈旧的 tab 动作缓存，仅剩「设置页改配置的同时正在筛选」这一极窄窗口；③ `filterByMoQi` 除 `StrokeCandidateWord` 外对候选类型没有豁免，预测 / 云端 / 非汉字候选走同一条 frontier 首字规则（已写入代码注释）。
>
> 行为澄清：辅助筛选触发键只在**存在候选列表**时才是特殊键；没有 composition、或 `AuxiliaryFilter=Disabled` 时它按字面输入（会输入一个反引号）。这是设计使然，实机验证时不要误判为缺陷。
>
> 已定性（探针批次 run `36243942929`：同一探针矩阵同时构建并运行 `upstream-61474bd` 与 `fork-0d0102b8`，两条腿输出**逐字相同**）：之前那条"剩余音节消失"的观察**不成立**。真实过程是"新字母被插入到光标处，导致未选中的输入被重新切分"——Shuangpin `xian` → 光标左移两次 → 选 西 后继续输入 `n`、`i`，preedit 为 `西ni an`（**剩余音节 `an` 始终在 composition 里**），只是首音节由 `an` 变成 `ni`，所以 `安` 不再位于候选前列；若**不移动光标**再续输入，候选列表仍含 `安`（`安妮,annie,安你,安,…`），与 Pinyin + 分隔符路径一致。依据：`im/pinyin/pinyincandidate.cpp:311` → libime `pinyincontext.cpp:607/297/672` 的选择路径，加上 libime `pinyincontext.cpp:465 typeImpl()` 的 `cancelTill(cursor())` 与 fcitx5 core `inputbuffer.cpp:79` 的**在光标处插入**。归类：upstream 既有语义 + 测试构造错误，**非本分支 regression**（新增代码从不修改 LibIME context），**不需要修改 LibIME 或产品代码**。详见 `research/shuangpin-cursor-selection.md`。

> 机制演进（含实测证据）：① 在 addon 里给 `install(FILES ...)` 加 `COMPONENT config` —— 失败（run `36130867729`）：`installLibraryConfig[...]` 只先构建 `generate-desktop-file` 就执行 `cmake --install --component config`，早于 addon 原生构建，构建期生成的码表此时尚不存在，`d7ec70b` 已 revert；② 改在 `fcitx5-android` 的 app CMakeLists 于 configure 阶段取表并以 `prebuilt-assets` 安装 —— 可行（run `36131495456`），但会给 Android 侧引入 MoQi 专属改动；③ 最终方案：addon 在 configure 阶段取表 + `COMPONENT config` —— 可行且 Android 零改动（run `36135660468`）。
>
> 附带结论：墨奇表的分发**不再需要 fork `fcitx5-android`**；stock fcitx5-android 只要使用本分支的 addon 子模块即可打包该表。Phase 7 评估长期 fork 必要性时应计入这一点。
>
> 上游贡献与长期 fork 边界评估：`research/upstream-fork-assessment.md`（四类改动归属、四个特定问题答复、发布复现性缺陷、快速 CI 回路评估）。

### 范围

- 完整固定 MoQi table 集成；
- Android 构建、安装和实际输入体验；
- Auxiliary Filter 配置体验；
- 用户学习、性能和稳定性；
- 评估向上游贡献及长期 fork 必要性。

> 已记录的打包发现：`fcitx5-android` 只安装 CMake 的 `prebuilt-assets` / `config` / `translation` 三个 component，addon 中无 COMPONENT 的 `install(FILES ...)` 不会进入 APK。Phase 2 的测试 APK 用 CI 临时步骤绕过，本 Phase 需要落地正式机制。

### Final Review（Phase 3 收口）

Phase 3 未定义独立的 "Exit Criteria" 小节（ROADMAP 中只有 Phase 2 与 Phase 4 各有一份），因此以本 Phase 的**目标 + 当前批次 + 范围**作为事实标准逐项核对：

| # | 验收项（来源） | 证据 | 状态 |
|---|---|---|---|
| 1 | 墨奇表经**正常构建**分发（目标） | run `36135660468`；`app/src/main/cpp/CMakeLists.txt` 与上游逐字一致；本机从 APK 复核 asset 路径与 SHA256 | ✅ |
| 2 | 码表 pin 单一来源与更新策略（批次 2） | `modules/pinyinhelper/moqima-gb18030.cmake`（`6d8ba8f1` + `66deab4a…`）；本机 cmake 3.31.6 实跑该段逻辑 | ✅ |
| 3 | Android 实机回归（批次 3 / 目标） | 项目所有者人工回报 12/12 通过（**人工证据，无截图/logcat**） | ✅ |
| 4 | 发布线：固定包名 + 自有密钥 + tag 触发 + **可复现**（批次 4） | `v0.1.3-moqi.2`；run `36158362313`（日志 `addon commit checked out: 0d0102b8…`）；APK 签名指纹与自有密钥一致；`v0.1.3-moqi.1` 未被改动 | ✅ |
| 5 | edge cases 补测与审计（批次 5） | run `36143794062`（ctest 9/9）；三条审计结论 | ✅ |
| 6 | 双拼 partial selection 问题定性（批次 6） | 探针批次 run `36243942929`（upstream 与 fork 输出逐字相同 → upstream 语义，非 regression）；据此恢复断言（`8ccb09f`，run `36246062225` 全绿） | ✅ |
| 7 | 上游贡献 / 长期 fork 评估（范围） | `research/upstream-fork-assessment.md` | ✅ |
| 8 | PR #1 状态 | OPEN / Draft / MERGEABLE，head `8ccb09f`，clang-format + gcc + clang + CodeQL 全 SUCCESS | ✅ |
| 9 | Auxiliary Filter 配置体验（范围） | 实机 12 项中的第 1、2、11 项（显示 / 持久化 / Disabled 行为） | ✅（人工证据） |
| 10 | 用户学习、性能和稳定性（范围） | **未系统验证** | ⚠️ 限制 |

**尚存限制**：

- 实机证据为**项目所有者人工回报**，无截图/logcat，不可自动复核；仅覆盖 arm64-v8a 正式包。
- "用户学习、性能与稳定性"未系统测试（Phase 3 范围项之一）。
- 上游贡献尚未实际提交（只完成评估与分类）。
- `.debug` 预发布线与正式线不互通（设计如此，对外说明需保留）。
- 一次性探针分支 `probe/shuangpin-cursor-boundary`（`8ea2ec0`）仍在远端，未合并、不影响 PR #1，待另行清理。

**Phase 4 入口**：以本 Phase 产物为基线（正式包 `v0.1.3-moqi.2`、addon `8ccb09f01da675eda53527136d07a977eb63320e`、Android fork `59efbf543d1ca47041886794e085cef703bde180`）进入 Phase 4 — Voice Input PoC；先确认 PoC 边界与设备/环境，不做 Provider framework 的过度设计。

## Phase 4 — Voice Input PoC

**状态：NOT STARTED**

优先复用：

- Fcitx5 Android 现有 microphone UI；
- upstream WIP SpeechRecognizer voice-input 工作；
- Android `SpeechRecognizer`；
- Android `RecognitionService`。

验证：

```text
Microphone
→ Voice Trigger
→ SpeechRecognizer
→ RecognitionService
→ ASR
→ Raw Transcript
→ IME
```

并验证：

```text
Long-press Space
→ same Voice Trigger
```

Exit Criteria：

- microphone 与可选 long-press Space 进入同一 voice path；
- permission/lifecycle/start/stop/cancel 正确；
- partial/final transcript 正确；
- Voice Trigger 不绑定 ASR vendor；
- ASR implementation boundary 明确；
- 数据流可审计。

## Phase 5 — ASR Provider Architecture / PoC

**状态：NOT STARTED**

根据 Phase 4 实际需求建立最小 Provider abstraction，验证代表性的：

- cloud ASR；
- OpenAI-compatible；
- local/self-hosted ASR；
- provider switching；
- streaming/non-streaming；
- 中文质量、延迟和数据流。

不在 Voice PoC 前过度设计 Provider framework。

## Phase 6 — Optional LLM Post-processing

**状态：NOT STARTED**

实现：

```text
ASR
→ Raw Transcript
→ Optional Text Post Processor
→ Final Transcript
```

要求可完全关闭、与 ASR Provider 独立、LLM Provider 独立配置，并明确发送文本和隐私边界。

## Phase 7 — Android Product Integration

**状态：NOT STARTED**

最终整合：

- Auxiliary Filter settings；
- Voice settings；
- ASR Provider settings；
- optional LLM settings；
- privacy/data-flow UI；
- packaging/release。

## Phase 8 — Additional Platforms

**状态：NOT STARTED**

Android 架构稳定后再评估 Windows、Linux、macOS、iOS，并保持 Trigger / Configured Implementation 分离。

## 工程节奏

- 修改前核对源码/API/现有测试；
- 逻辑完整的小批次开发；
- push 前 diff review、格式/静态检查和适用本地测试；
- GitHub Actions 仅作阶段性集成验证；
- 不通过反复提交猜测性修复；
- docs-only 原则上不触发重型 CI。

## 当前下一步

**Phase 3 已完成**（证据、限制与 Phase 4 入口见 Phase 3 的 Final Review）。进入 **Phase 4 — Voice Input PoC**：

- 入口定义见下方 Phase 4 区块：microphone 与可选 long-press Space 走同一 voice path，复用 upstream WIP SpeechRecognizer 与 Android `SpeechRecognizer` / `RecognitionService`；
- 基线：正式包 `v0.1.3-moqi.2`、addon `8ccb09f01da675eda53527136d07a977eb63320e`、Android fork `59efbf543d1ca47041886794e085cef703bde180`；
- 前置：先确认 PoC 边界与设备/环境，再决定实现范围；不在 Voice PoC 前过度设计 Provider framework。
- 并行可选（不阻塞 Phase 4 启动）：上游贡献落地（`research/upstream-fork-assessment.md` 第 A 类三项）；一次性探针分支清理；快速 CI 回路（此前暂缓）。
