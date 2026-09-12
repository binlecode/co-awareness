# CLAUDE.md / AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) and all coding agents working in this repository.
`AGENTS.md` 与 `CLAUDE.md` 互为符号链接，保持单一正本，编辑任意一个即可。
保持精炼与高信噪比；详细架构细则、演进历史与外部调研统一归入 `docs/`。

---

## ⚠️ 关键陷阱 —— 开工前必看

开工前必须明确以下六条硬底线，**违反会导致进程崩溃、构建并发冲突或破坏核心设计哲学**：

- 🔴 **单文件无包架构（Unbundled Swift）**：所有业务逻辑集中于 `MenuBarLoadRunner.swift`，零外部包依赖，由 `swiftc -O -strict-concurrency=complete` 编译；绝不引入 Xcode 项目、SwiftPM (`Package.swift`)、CocoaPods 或外部第三方库。类均标注 `@MainActor`，构建必须保持零警告。
- 🔴 **原子重命名编译（Atomic Rename）**：启动脚本 `--precompile` 必须先编译到临时路径再通过 `mv`（`rename(2)`）原子替换目标二进制；严禁在运行中原地覆盖 Mach-O 二进制文件，否则破坏正在运行进程的内存分页导致当场崩溃。
- 🔴 **单例守卫必须前置于编译（Singleton Guard Before Compile）**：启动器中的 `pgrep -U "$(id -u)"` 检查必须严格在 `compile_if_stale` 之前执行，防止多个并发启动请求同时触发 `swiftc` 写入同一目标路径。
- 🔴 **零 Mock 真实驱动测试（Zero Mocks / Real Binary Assertions）**：`tests/qa.sh` 是唯一测试套件，必须驱动真实二进制检验真实副作用；严禁在测试中复制业务代码类型；只允许使用无侵入可观测性环境变量（`EXIT_AFTER`, `LOG_*`, `FORCE_BATTERY`, `STATE_FILE`），严禁引入改变业务决策逻辑的 hook。
- 🔴 **无特权只读遥测与纯用户态（Unprivileged & Read-Only）**：遥测仅限公开或无特权的 Mach / IOKit / SMC 接口，严禁请求 root，严禁通过 `pmset disablesleep` 修改系统全局 NVRAM 电源策略；Keep Awake 仅通过绑定自身 PID 的 `caffeinate -di -w <pid>` 实现，保证进程异常退出时内核自动回收。
- 🔴 **5% 硬件电池底线不可逾越**：电量 ≤ 5% 时强制释放 Keep Awake，任何 CLI 参数、状态恢复或菜单操作均严禁绕过此硬底线。

---

## 🔴 第一条：不要自作聪明

**有疑问或做技术选型时，严格按两步执行，顺序不可颠倒：**

1. **先 grounding** —— 查真实代码（`MenuBarLoadRunner.swift`、`menubar-load-runner`）、核查真实运行输出与可观测性探针（`MENUBAR_LOAD_RUNNER_EXIT_AFTER`, `LOG_*`），深入代码实现与真实终端输出，**严禁凭空假设向下推演**。
2. **再确认** —— 严禁静默新增/废弃 CLI 参数、修改 `state.json` 数据结构、私自放宽安全门控或重新解释既有设计。若实测推翻了前提，如实报送发现并提问，**不要自行改动设计范围或重排优先级**。

---

## 项目性质

**MenuBar Load Runner** —— 原生 macOS 状态栏动态负载可视化与轻量级诊断监控套件（Swift + AppKit）。

- **单文件 + 原生启动脚本**：`MenuBarLoadRunner.swift` (~6.6k 行) + 原生 zsh 启动脚本 `menubar-load-runner`，无需 Xcode / SwiftPM，直截了当。
- **动态帧率自适应**：9 种无特权硬件遥测源（CPU、内存+Swap、GPU、网络、磁盘、风扇、电池放电电流、芯片结温、神经引擎功耗）驱动状态栏 GIF 变速播放。
- **平滑归一化与自限流**：移植自 `btop` 的 `ThroughputScaler` 自适应非对称迟滞缩放无界速率；高热/低电量/内存压力下自动减半自身帧率；全遮挡（刘海/隐藏/灭屏）0% CPU 暂停；支持系统 Reduce Motion 与手动 Freeze（冻结时读数自动交接给标签栏）。
- **进程生命周期防休眠**：内置基于 `caffeinate -di -w <pid>` 的 Keep Awake 与 `IOPMCopyAssertionsByProcess` 外部断言嗅探器。

---

## 文档分工（先读，别重复摸索）

全部架构与生命周期文档统一位于 `docs/`。根目录只留 `README.md`（人的入口）和 `CLAUDE.md` / `AGENTS.md`（Agent 指令）。

- 🔴 **全文档唯一正本路由在 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) §0**，这是架构伞文档。
- 🔴 **未决议题、待做特性与记录在案的 NO 只认 [`docs/ROADMAP.md`](docs/ROADMAP.md)**。
- 🔴 **已建成架构细则正本在 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)**。本文不维护第二份产品设计/架构细则。

### 文档命名与 ADLC 流水线硬规矩

| 文种 | 命名规范 | 职责与生命周期 |
|---|---|---|
| **路线图** | `docs/ROADMAP.md` | 悬着的议题总表（记录在案的 NO、候选议题 `R<n>`、排期）；只记决定与条件，不记流水账；议题落成即移出 |
| **计划书** | `docs/PLAN-<topic>.md` | 单项特性的设计与实施草案；**主体做完蒸馏入 `docs/ARCHITECTURE.md` 后当场 `git rm` 删除**（绝不留存归档） |
| **架构正本** | `docs/ARCHITECTURE.md` | 系统已建成架构的正本（why 与 how，不重复代码中的 what）；包含 §0 路由表、系统拓扑、子系统规约与参数基线 |
| **外部调研** | `docs/RESEARCH-<topic>.md` | 外部竞品与业界生态调研及实测数据（**不入 ADLC 链**，不 land、不改名、不退休，供决策参考，测量过时后清理或重测） |

- **严禁新建文种**：没有 `REPORT-`、`DESIGN-`、`TODO-`、`LESSONS`、`RUNBOOK-` 或 `JOURNAL`。
- **系统结构图硬规则**：一律使用纯 ASCII 字符（`+ - | = v ^ < >`）绘制，严禁使用制表符（`┌─│`），对齐严格按 CJK 双倍字宽计算。
- **修改文档正文一律使用编辑工具（Edit）**，严禁使用 `sed` 破坏文档排版。
- **一个事实只在一处声明 (One fact, one place)**：文档严禁复制代码中已有陈述的 what；文档只记录 why 与 how，并跨文件交叉引用。

---

## 常用命令

```bash
# 启动与运行
./menubar-load-runner                       # 默认预设 (horse-white)，后台脱离终端运行
./menubar-load-runner --foreground           # 前台运行（查看 stderr / print 输出）
./menubar-load-runner dog-black --label value   # 指定预设 + 状态栏数值标签
./menubar-load-runner --help

# 编译与无启动检查
./menubar-load-runner --precompile           # 仅当源码较新时原子编译，不启动（保持运行实例不损坏）
swiftc -O -strict-concurrency=complete MenuBarLoadRunner.swift -o tmp/mblr-check  # 快速编译检查

# 自动化测试与调试钩子（均无需 TCC / 辅助功能权限）
MENUBAR_LOAD_RUNNER_EXIT_AFTER=5 ./tmp/mblr-check --load-source memory           # 运行 5s 自行退出 (exit 0)
MENUBAR_LOAD_RUNNER_FORCE_BATTERY=15:battery ./tmp/mblr-check --keep-awake 30m   # 模拟低电量 / 电池状态
MENUBAR_LOAD_RUNNER_LOG_SLOTS=1 ./tmp/mblr-check --label value 2>&1 | grep SLOTS # 打印状态栏槽位屏幕几何与宽度
MENUBAR_LOAD_RUNNER_LOG_ASSERTIONS=1 ./tmp/mblr-check 2>&1 | grep ASSERTIONS     # 打印过滤与防抖后的外部睡眠断言
MENUBAR_LOAD_RUNNER_LOG_AWAKE=1 ./tmp/mblr-check 2>&1 | grep AWAKE               # 打印睡眠阻止综合判定与菜单行文本
MENUBAR_LOAD_RUNNER_LOG_ANIMATION=1 ./tmp/mblr-check 2>&1 | grep ANIM           # 打印动画冻结状态与游标

# 自动化测试套件
tests/qa.sh --core                           # 核心门禁（CI 友好，不依赖 WindowServer，秒级）
tests/qa.sh                                 # 全量回归测试（需要活跃 WindowServer / GUI 会话）
tests/qa.sh --launcher                       # 包含启动器单例与原子替换破坏性测试（会 pkill 实例）

# 开机自启 LaunchAgent（基于 scripts/ 已有脚本，勿手写 plist）
./scripts/install-login-item.sh [preset] [flags]   # 安装并启动用户 LaunchAgent
./scripts/uninstall-login-item.sh                  # 卸载 LaunchAgent

# 进程清理
pkill -f 'MenuBarLoadRunner'                 # 停止当前用户正在运行的实例
```

---

## 架构要点（正本见 docs/ARCHITECTURE.md）

- **9 种无特权硬件遥测源** (`docs/ARCHITECTURE.md` §4)：CPU (Mach)、Memory + Swap (Mach `vm_statistics64`)、GPU (IOAccelerator)、Network/Disk (IOKit 计数器增量)、Fan RPM (SMC)、Battery mA (IOKit PS)、Max Die Temp (SMC 二分查找 `Tp**`/`Tpx*` 传感器集群最大值)、ANE Watts (`IOReport` "Energy Model" 订阅式增量采样，唯一的私有 API，经 `dlopen`/`dlsym` 运行时绑定；正本见 §4.5)。每种源均具备 `isAvailable` 探测，不可用时平滑降级。
- **ThroughputScaler 速率归一化** (`docs/ARCHITECTURE.md` §4.2)：无界速率（网速/磁盘/swap/电池电流）经自适应滑动窗口归一化到 0..1，双向非对称裕量 + 迟滞计数器防抖；有界百分比与绝对温度映射不走 Scaler。
- **CADisplayLink 与自限流** (`docs/ARCHITECTURE.md` §3, §5)：屏幕刷新率同步的 vsync 游戏循环；全遮挡（刘海/隐藏/灭屏）时完全暂停渲染（0% CPU）；高热/低电量/内存压力下自动减半自身帧率；尊重系统 Reduce Motion 与手动 Freeze（冻结时读数自动交接给标签栏，R17）。
- **Keep Awake 睡眠阻止与外部断言嗅探** (`docs/ARCHITECTURE.md` §7)：通过 `SleepPreventer` 启动 `caffeinate -di -w <pid>` 绑定进程生命周期；支持预设/自定义定时窗口（跨重启恢复）；底线 5% 电池保护；通过 `IOPMCopyAssertionsByProcess` 嗅探系统其他进程断言，两段式归因排布（This Mac vs This App）。
- **双槽位状态栏标签模型** (`docs/ARCHITECTURE.md` §6)：标签采用独立状态栏项而非在 GIF 上烘焙文字；预建左右两个槽位（`labelItemLeft`, `labelItemRight`）以克服 macOS 状态栏槽位不可重排限制，严格采用花样空格（U+2007）预占位防抖。
- **状态持久化单点守恒** (`docs/ARCHITECTURE.md` §8.2)：`~/Library/Application Support/menubar-load-runner/state.json` 由 `persistState()` 统一全量写盘，持久化意图（Intent）而非易失运行状态。
- **预设注册表与自更新** (`docs/ARCHITECTURE.md` §9)：动图配置完全由 `gifs/presets.json` 驱动；自更新先 `git pull --ff-only` 再执行 `--precompile` 原子编译，最后在弹窗提示后由 detached 脚本完成重启。

### 添加内置预设流程
1. 将优化裁剪好的 GIF 放入 `gifs/<name>.gif`（必须透明背景、紧凑边界）。
2. 在 `gifs/presets.json` 的 `presets` 数组中添加配置项（指定 `key`、`menuTitle`、`file`、`speed` 范围与指数）。
3. 运行 `./menubar-load-runner <name>` 验证宽高比适配与各档位动画速度。
4. 运行 `tests/qa.sh --core` 确认解析无误。

---

## 改动时的红线

动手改动代码前必须确认以下八项硬约束：

1. **严禁引入构建系统或外部运行时依赖**：坚守单文件 Swift 架构与原生启动脚本，`-strict-concurrency=complete` 必须保持零警告。
2. **严禁破坏无特权只读原则**：遥测只能通过系统公开/免提权接口读取；严禁请求 root，严禁通过 `pmset` 篡改系统全局电源配置。
3. **严禁在测试中引入 mock 或行为注入**：测试必须通过 `tests/qa.sh` 驱动真实二进制；禁止在测试中复制业务代码类型；禁止新增改变业务决策路径的测试 hook。
4. **严禁原地覆盖正在运行的二进制**：预编译与更新流程必须编译到临时文件后通过 `mv` 原子重命名替换，防止破坏运行中进程的内存分页。
5. **单例守卫必须前置于编译**：启动脚本必须在编译前执行 `pgrep -U "$(id -u)"`，防止多实例并发编译写入同一输出。
6. **Keep Awake 5% 电池保护底线不可逾越**：电量 ≤ 5% 必须无条件释放断言，任何 CLI 参数、状态恢复或菜单点击均严禁绕过此底线。
7. **严禁破坏状态持久化单写者模型**：`state.json` 的写盘只能由 `persistState()` 统一驱动，禁止在局部 observer 或高频采样循环中自行写盘。
8. **一个事实只在一处声明**：详细架构机制与参数一律维护在 `docs/ARCHITECTURE.md`；改动代码后必须同步更新 as-built 文档；落地后立即删除对应的 `PLAN-*.md`。

---

## 测试与回归约定

- **驱动真实二进制**：`tests/qa.sh` 是唯一的自动化回归测试套件，分级运行：`--core`（语法、编译、CLI 解析与版本基线，无 GUI 依赖）、默认（包含 GUI 状态栏与断言检查）、`--launcher`（启动器单例与并发测试）。
- **只使用无侵入可观测性钩子**：`MENUBAR_LOAD_RUNNER_EXIT_AFTER`（生命周期截断）、`FORCE_BATTERY`（模拟电量）、`LOG_SLOTS` / `LOG_ASSERTIONS` / `LOG_AWAKE` / `LOG_ANIMATION`（日志输出内部判定）。禁止任何改变业务决策逻辑的 hook。
- **环境无法测定时输出 NOTE，绝不造假 PASS/FAIL**：例如屏幕拥挤、无电池桌面机、系统自带睡眠断言等外部不可控状态，如实输出 NOTE。
- **严禁谎称覆盖**：测试无法测定的系统边界必须诚实声明，绝不引入虚假断言。

---

## 环境与规范

- **运行平台**：macOS 14+ 完整支持（macOS 12/13 降级为 60Hz Timer）。
- **语言与并发**：Swift 5 模式，`-strict-concurrency=complete` 零警告；UI 与状态强绑定 `@MainActor`。
- **启动器与脚本**：严格保证 `/bin/bash` 3.2 与 macOS 原生 `/bin/zsh` 兼容性。
- **版本规范**：遵循 SemVer 2.0.0，以 Git Tag（`vX.Y.Z`）驱动更新发现与发布。
