# EnergyOS（动态精力调度）

基于「精力槽」与任务切换的智能时间管理应用，**Flutter** 跨平台（含 Web 调试）。

| 文档 | 说明 |
|------|------|
| [docs/DESIGN.md](docs/DESIGN.md) | UI 设计体系（The Biophilic Editor：配色、字体、无分割线、玻璃底栏等） |
| [docs/PRD.md](docs/PRD.md) | MVP 产品范围（若与实现有出入，以代码与本文为准） |
| [docs/EnergyOS_产品技术文档.md](docs/EnergyOS_产品技术文档.md) | 产品设计、节律、事件流、SQLite、精力引擎与游戏化 |
| [GROWTH_VALIDATION.md](GROWTH_VALIDATION.md) | 增长验证计划（三个轻量实验） |
| [VALUE_PROPOSITION.md](VALUE_PROPOSITION.md) | 价值主张初稿 |
| [异步沟通.md](异步沟通.md) | 开发待办与测试清单（开发者内部） |

- **Flutter 应用工程**：[`app/`](app/)（业务代码在 `app/lib/`）
- **本机自带的 Flutter SDK（可选）**：[`flutter_sdk/`](flutter_sdk/)（体积大，已加入 `.gitignore`）

---

## 近期进展（摘要）

**截至 2026-03-29**（逐条流水见 [异步沟通.md](异步沟通.md) 末尾「进度更新」）

| 方向 | 内容 |
|------|------|
| **观测与增长** | 接入 **Firebase Analytics**（`AnalyticsService`、10 个自定义事件、`FirebaseAnalyticsObserver` 自动 `screen_view`），便于 T+1 对照真实使用复盘。 |
| **任务与日程** | **v0.1.1**：暂停/恢复专注、**继续专注**卡片与**累计时钟**；「今天」**左滑**改/删；**跨日**归档昨日任务；添加/修改任务弹窗（400ms/300ms 动画）；时间轴竖线、推荐点击乐观 UI 等交互打磨。 |
| **健康打断** | **低精力**全屏弹窗（&lt;20% focus）；**久坐 45 分钟**独立全屏弹窗（绿色系、4 类活动、Riverpod 记已提醒任务）。 |
| **AI 与画像** | memory.md、结束今天日报、周日/调试周报、历史报告、大模型设置（测试连接、强制周报）等与 **v10** 技术文档第九章一致。 |
| **首页精力** | 圆环 **近满格弧长映射**（`ringVisualProgressRatio`）便于看出 100%→99%；环两侧 **±10%** 写入 `manual_adjust`；线宽与视觉微调。 |
| **我的 / 节律图** | 当前小时仅保留**下指箭头**（去掉「此刻」文案），箭头尺寸随字阶。 |
| **工程交付** | **Android**：`flutter build apk --release` 写入 README；`key.properties` 中 `storeFile=../../../energyos.jks` 指向仓库根 `energyos.jks`；`gradle.properties` 中 **`kotlin.incremental=false`** 缓解工程在 D:、Pub 在 C: 时的 Kotlin 缓存报错。**Web**：`--no-web-resources-cdn`、**固定 `--web-port`**（IndexedDB 按端口隔离）说明完备；`app/.vscode/launch.json` 提供无 CDN 调试配置。 |

---

## 已实现功能（与当前代码对齐）

### 基础数据层（DB 版本 10）

- **SQLite** 全表结构 + `Repository` 抽象；Web 使用 `sqflite_common_ffi_web`（主线程，`noWebWorker: true`）
- **精力模型（时间等效）**：1 格 = 1 小时容量；总格数 5～8 格（上限 8）；主界面以**百分比**显示；**无「工作时间窗口」裁剪**
- **节律**：`rhythm_curve.arousal_level` 整数三态（0 困倦 / 1 平稳 / 2 清醒），四种 chronotype 模板（早晨 / 上午 / 下午 / 晚间）
- **任务类型（两档）**：`focus`（消耗精力）/ `restore`（恢复精力）；旧档字符串在 `parseTaskType` 中映射
- **事件流**：`task_start` / `task_done` / `manual_adjust` / `rest_start`，支持 `pair_id` 虚拟完成对

### 功能页面

- **模式选择（首次启动）**：体验模式（内存 SQLite，关闭清空）/ 本地存储（持久化）
- **Onboarding**：最清醒时段四选一，写入节律 + 种子任务
- **「现在」屏**：精力环（256px）、当前任务区（进行中 / 推荐 / 空态）、手动 ±10% 精力调节；结算后显示庆祝卡片和「该去休息了」状态文字；精力 < 80% 且有 focus 任务进行中时，任务卡片显示「我需要休息一下」按钮，弹出恢复活动选择底栏（主动触发，可关闭）
- **「今天」屏**：时间轴视图（从 8:00 起），已完成 / 进行中 / 待开始三区；5 秒 tick 刷新进行中时长；长按删除未开始任务；**左滑**滑出「修改」（蓝）/ 「删除」（红）操作按钮
- **跨日任务清理**：新的一天开始时，归档所有旧任务（`is_archived = 1`），当天从空白开始，不再携带昨天的待办
- **添加 / 修改任务**（底部弹窗）：备注、任务名、精力类型二选一；备注写入 `tasks.notes`；修改时预填当前内容，弹窗动画 400ms 进场 / 300ms 退场
- **「我的」屏**：三区块布局——「节律」（当前节律类型只读展示 + 24 小时节律图）、「精力成长」（当前格子数 + 累计积分与解锁进度）、「设置」（大模型设置 / 精力画像 / 历史报告 / 切换数据模式）；格子数仅由结算积分自动解锁，不可手动调整

### 低精力全屏弹窗

精力跌破 20% 且有 focus 任务进行中时触发（5 秒检测间隔，恢复至 35% 后重置）：

- 情境化建议文字（<30 min / 30–90 min / >90 min 三档）
- 随机精力小知识（15 条循环）
- 6 种预设恢复活动（2 列网格），点击一键开始
- Ghost 按钮「继续当前任务」

### 久坐提醒全屏弹窗

focus 任务持续进行满 **45 分钟**时触发，与低精力弹窗在视觉和文案上完全区分：

- **绿色 pill**（vs 低精力红色），显示「已专注 X 分钟」
- 正文：提示久坐减缓血液循环、5–10 分钟起身活动即可重置身体
- 久坐小贴士：每坐 45 分钟起身一次可降低心血管风险
- **4 种活动推荐**（散步 / 拉伸 / 远眺 / 家务），偏向走动和护眼
- Ghost 按钮「我待会儿再动，继续工作」
- 同一任务只弹一次：alerted taskId 存入 `_prolongedFocusAlertedTaskIdProvider`（Riverpod），Tab 切换不会重置

### 每日结算

`SettlementService.settleDay()` 计算 `qualityScore`（0–100），四维度：

| 维度 | 满分 |
|------|------|
| 节律匹配（focus 任务在清醒时段） | 40 |
| 精力交替（focus / restore 穿插） | 25 |
| 未归零（精力未到 0） | 20 |
| 空转估算（无记录间隔 < 60 分钟） | 15 |

积分累计达到门槛（当前格数 × 350）自动解锁下一格精力。

### AI 功能（v10）

**配置**：用户在「我的 → 大模型设置」填入 DeepSeek API Key，Key 仅存本地。

**结束今天**（首页底部卡片）：

1. 运行结算，计算今日 6 项统计（专注次数、平均专注时长、恢复次数、平均恢复时长、计划完成率、任务切换次数）
2. 从 `memory.md` 读取用户精力画像，注入 AI system prompt 作为上下文
3. 调用 DeepSeek 生成 100 字以内陪伴文字（开头空两格；未配置 Key 时本地兜底）
4. 将今日统计写入 `memory.md` 的「近期数据摘要」section（同一天多次结算覆盖，不重复追加）
5. 弹出**日报卡片**：今日得分 + 统计网格 + AI 文字

**周报**：汇总 7 天日报数据，AI 返回 JSON（`summary` / `suggestion` / `chronotype_suggestion` / `pattern`）；将观察到的规律写入 `memory.md`；用户确认后将建议写入 memory 并更新 `users.chronotype`。

  若今天是**周日**且本周周报未生成，结算后自动触发；也可在「大模型设置 → 调试工具」中强制生成（生成后直接弹出 WeeklyReportSheet，可测试「确认调整」流程）。

**历史报告**（「我的 → 历史报告」）：日报 / 周报 Tab，各保留最近 30 / 12 条；点击卡片弹出完整内容。

### 精力画像（memory.md）

`MemoryService` 维护一个结构化 Markdown 文件（本地存储或 SQLite 字段），作为 AI 的持久化上下文：

| Section | 来源 |
|---------|------|
| 基本信息 | Onboarding（节律类型、精力格子数） |
| 近期数据摘要 | 每次结算自动追加（保留最近 14 天） |
| 观察到的规律 | 周报生成时写入 AI 提炼的 pattern |
| 已确认的建议 | 用户点击「确认调整」后写入 |

- **「我的 → 精力画像」**：可查看渲染版 / 原始 Markdown，一键复制到剪贴板（可粘贴到任意 AI 工具）
- **平台存储**：Native → 本地文件 `energyos_memory.md`；Web → `ai_config.memory_content` 字段

### 数据分析（Firebase Analytics）

- **依赖**：`firebase_core` + `firebase_analytics`；启动时 `Firebase.initializeApp`（配置见 `app/lib/firebase_options.dart`）。
- **统一入口**：`app/lib/core/services/analytics_service.dart` 的 **AnalyticsService**——业务代码只调该类，不直接调 `FirebaseAnalytics`。
- **自定义事件（10 个）**：**激活漏斗** — `onboarding_completed`，`task_added`，`focus_started`（含 `task_type` / `energy_ratio` / `hour_of_day`），`focus_completed`，`day_ended`；**功能价值** — `rest_break_requested`，`daily_report_viewed`，`weekly_report_confirmed`，`memory_viewer_opened`，`ai_configured`。
- **导航**：`MaterialApp` 注册 `FirebaseAnalyticsObserver`，自动上报 `screen_view`。
- **控制台**：Firebase 项目内可查看 **日活、留存**、上述事件及参数；用于 **T+1** 起对照真实使用做复盘（数据延迟以 Firebase 报表为准）。

---

## 常见问题

### `Failed to fetch` / `canvaskit.js`

加 `--no-web-resources-cdn` 参数运行，或使用 VS Code 的 **`Flutter Web (Chrome, no CDN)`** 配置。

### Web 上 SQLite 空白页 / `sqflite_sw.js` 报错

重跑 `dart run sqflite_common_ffi_web:setup`，确认 `app/web/` 下有 `sqflite_sw.js` 与 `sqlite3.wasm`。

### `WebAssembly.instantiate(): Import #25 "env"`

三者必须对齐：`sqflite_common_ffi: 2.3.5` + `sqlite3: 2.4.6`（`pubspec.yaml` 的 `dependency_overrides` 已固定）+ `web/sqlite3.wasm`（来自 setup 或手动下载 sqlite3-2.4.6 版本）。

### Web 数据为什么每次重启都清空

Web 端使用 `sqflite_common_ffi_web` 的 `noWebWorker: true` 模式，该模式下 SQLite 运行于主线程内存，不写入 IndexedDB。首次启动选择「本地存储」模式即可持久化（数据写入浏览器 IndexedDB）。

---

## 项目结构速览

```
app/lib/
├── core/
│   ├── bootstrap/       day_bootstrap（跨日重置）
│   ├── content/         restore_content（恢复活动库 + 小知识）
│   ├── database/        database_helper + database_constants
│   ├── energy/          energy_calculator（精力推演纯函数）
│   ├── recommend/       recommendation（旧，暂保留）
│   ├── rhythm/          rhythm_templates
│   ├── services/        energy_runtime / settlement_service /
│   │                    daily_summary_service / weekly_report_service /
│   │                    deepseek_client / memory_service / analytics_service
│   ├── storage/         memory_storage（平台条件导出：native文件/web SQLite）
│   ├── task/            task_flow
│   ├── theme/           app_theme
│   ├── today/           today_task_list_builder
│   ├── utils/           time_utils / energy_display
│   └── widgets/         primary_gradient_button
├── features/
│   ├── now/             NowScreen + widgets
│   ├── onboarding/      OnboardingScreen / ModeSelectionScreen
│   ├── profile/         ProfileScreen / AiSettingsScreen / MemoryViewerScreen
│   ├── reports/         DailyReportSheet / WeeklyReportSheet /
│   │                    ReportsHistoryScreen
│   ├── shell/           MainShell（底栏导航）
│   ├── stats/           StatsScreen（暂未接入导航）
│   └── today/           TodayDayScreen + widgets
├── models/              所有数据模型
├── providers.dart       全局 Riverpod Provider
└── repositories/        所有 Repository 实现
```
