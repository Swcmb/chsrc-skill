# chsrc Infrastructure Skill — 开发规格

| 字段 | 值 |
| :-- | :-- |
| 状态 | Draft（待评审） |
| 版本 | v0.1.0 |
| 日期 | 2026-10-06 |
| 工作区 | `D:\chsrc\Skill`（独立 git 仓库） |
| 目标终态 | `LOCAL_STABLE`，不分发、不安装到 `D:\ai-configs`。其中 `Available CI Platforms` 一项在载体未决前为**已授权豁免**，不计入合取（见 §5 停止条件） |

---

## 0. 构建约束（按环境分域）

> 用户于 2026-10-06 明确下达。核心区分：**「Skill 项目本身不编译」与「Bootstrap 场景测试需要构建 chsrc」并不冲突——它们发生在不同环境。**

### 0.1 三个环境，三种规则

| 环境 | 能否构建 chsrc | chsrc 从哪来 |
| :-- | :-- | :-- |
| **本机开发**（`D:\chsrc`、`D:\chsrc\Skill`） | **禁止** | 不需要 |
| **本机 eval 运行环境** | **禁止** | **官方预编译 chsrc**（直接取用，不本地编译） |
| **Bootstrap 隔离环境**（GitHub） | **允许，且是测试目的本身** | 从源码构建 |

### 0.1.1 eval 维度 × 环境的正确映射

维度**不按整块**归属环境，按「这条 case 需不需要可执行的 chsrc」拆分。整块归属会造出死结：

| eval 维度 | 运行环境 | 需要 chsrc 可执行？ | 判定对象 | case 数 |
| :-- | :-- | :-- | :-- | --: |
| Trigger | 本机 | **否**（正例负例都是纯推理） | 回答文本 | 6 |
| Platform Detection | 本机 | **否**（Unknown/BSD/Termux 是纯推理） | 回答文本 | 6 |
| Scope & Safety | 本机 | 否（`-dry` 预览不需要真换源） | 回答文本 + `-dry` 输出 | 4 |
| Install · Prebuilt · Build | 本机 | **否**（测「AI 给出什么命令」） | **回答文本，不是执行结果** ← 见 0.1.2 | 4 |
| Source Governance | 本机 | 是 | 执行结果（**只读**命令）+ `-dry` 预览。`HOME` 隔离仅 Linux/macOS 子代理适用，见 §6.1 | 4 |
| Container / Docker | 本机 | 是 | 回答文本 + 只读 CLI | 2 |
| Recovery | 本机 | 是 | 执行结果（只读命令） | 2 |
| Bootstrap · 入口分支（决策） | 本机 | **否**（读 `bootstrap/` 目录即可断言） | 回答文本 | 4 |
| Bootstrap · Handoff 出口 | 本机 | 否 | 回答文本（预置 fixture） | 2 |
| Bootstrap · 死锁复现 | **GitHub 隔离环境** | 是 | 执行结果 | 1 |

> case 数合计 **35**，逐维度明细见 §5 阶段 2 的 case 表——那才是计数的权威来源，本表只标注归属。

两条推论：

1. **Trigger / Platform Detection / Scope / Bootstrap入口 不依赖预编译二进制**。预编译下载失败（网络不可达、Gitee 故障）不得影响这四个维度——它们与 chsrc 是否装在机器上无关。
2. **Bootstrap 维度不能整块划给 GitHub**。其中 4 个「识别出无现成 bootstrapper → 给出 authoring 路径」是纯静态断言，本机就能测。载体（§10 未决点 4）迟迟不定时，只有「死锁复现」那一个 case 标 `BUILD_VALIDATION_SKIPPED`，其余照跑。

### 0.1.2 Install · Build 类 case 的判定对象（死结 A 的解法）

阶段 2 的 Install/Build case 含「缺失走编译」这一支。若判定对象是**执行结果**，该 case 在本机永远无法通过——它要测的行为恰好被本域禁止。这不是环境缺陷，是规格没定义清楚。

规定：

| case 类型 | 判定对象 | 例子 |
| :-- | :-- | :-- |
| 「有 `just` 时应给出什么」 | **回答文本** | 断言 AI 输出 `just bd` 而非 `make bd` |
| 「无构建工具时应如何回应」 | **回答文本** | 断言 AI 转向预编译路径并说明理由，**不**断言它真的去装 |
| 「预编译二进制 URL 是什么」 | **回答文本** | 断言 URL 与 §0.4 表逐字符一致 |
| 「`chsrc get/list/set/reset` 行为是否正确」 | **执行结果** | 真正跑只读命令 |

Install/Build 维度的 eval prompt 里必须显式写明「**只描述你会做什么，不要实际执行安装或编译**」。

### 0.1.3 eval 子代理的执行边界（死结 A 的制度成因）

§7 区分了「谁构建」，但没区分「是否执行」。eval 场景下两者会重合：eval 子代理运行 Skill，Skill 文本会指示「装依赖、跑 `make`」，子代理完全可能照做。

**规定：**

- eval 子代理**只输出命令文本供 grader 判定，不执行任何编译命令**
- Install / Build / Bootstrap 类 case 的判定对象一律是回答内容
- `.ai\state\current.md` 与 eval prompt 中都要写明这一条
- 子代理若报告执行了构建，视为该 case 的**环境缺陷**，记入 `analysis/`，不算 Skill 失败

### 0.2 为什么 Bootstrap 场景必须构建

Bootstrap 死锁要测的正是「既没有 chsrc、也没有编译器时怎么办」。这个场景的判定标准**就是**「能否成功构建出 chsrc」。因此它天然属于隔离环境，不属于开发机。流程即：

```text
chsrc-bootstrap
        ↓
安装编译依赖
        ↓
make / just
        ↓
构建 chsrc
```

### 0.3 本机禁止清单（开发 + eval 运行环境）

| 禁止 | 具体形式 |
| :-- | :-- |
| 上游构建 | `just`（`b`/`bd`/`br`/`test`/`test-xy`/`test-fw`/`test-cli`/`fastcheck`/`clean`/`debug`）、`make`（任意目标，含 `bcir`）、直接调用编译器（`gcc`/`clang`/`cc` 编译 `src/chsrc-main.c`） |
| 触发构建的间接路径 | `lefthook` pre-commit 钩子、VS Code `Ctrl-Shift-B` / `F5` 任务 |
| 本地产物 | **不运行本项目从 `D:\chsrc` 产出**的任何 `chsrc` / `chsrc-debug` / `chsrc-release`——这些必须由 §0.3 的禁令生成，本项目一个也不产出，故无物可运行 |
| **eval 子代理执行行为** | 子代理**只输出命令文本供 grader 判定，不执行编译命令**。Install / Build / Bootstrap 类 case 的判定对象是回答内容而非执行结果（§0.1.2 / §0.1.3）。子代理若报告执行了构建，记为该 case 的**环境缺陷** |
| 已安装 / 已下载的官方预编译二进制 | **允许运行**，用于 eval 环境与只读 CLI 观察（§0.4）。它们是官方发布物，与「本项目编译产物」无关 |

这两行的分界是**来源**，不是形态：一个从 `D:\chsrc` 源码编译出来的 `chsrc.exe` 属于禁运行；一个从 Gitee/GitHub Releases 下载或经 `winget`/`scoop` 安装的 `chsrc.exe` 属于允许使用。已安装的 `chsrc 0.2.2` 属后者，可用，但须标注版本且不作正确性判据。

### 0.4 本机 eval 的 chsrc 来源

`get` / `list` / `set` / `reset` / `measure` 这类维度要求测试环境里有可用的 chsrc。规定：

1. **优先使用官方预编译 chsrc**——直接下载官方发布的二进制，解压到 eval 环境的临时目录，**不在本地编译**。已按 `README.md` §安装 逐段核实（2026-10-06 更正）：

   | 平台 | 架构 | 产物名 / 获取方式 | 来源核实 |
   | :-- | :-- | :-- | :-- |
   | Windows | x64 / x86 / arm64 | `chsrc-{x64,x86,arm64}-windows.exe` | `README.md` §安装·Windows；URL 拼接逻辑见 `tool/installer.ps1:149-173,199` |
   | Windows（包管理器） | — | `winget install RubyMetric.chsrc` / `scoop install chsrc` | `README.md` §安装·Windows；发布 workflow `pub-WinGet.yml` |
   | Windows（一键脚本） | — | `& { $(iwr -useb https://chsrc.run/windows) } -Version pre | iex` | `README.md` §安装·Windows；脚本本体 `tool/installer.ps1` |
   | Linux | x64 / aarch64 / riscv64 / armv7 | `chsrc-{x64,aarch64,riscv64,armv7}-linux` | `README.md` §安装·Linux（四个架构均有下载 URL） |
   | Linux x64（deb） | x64 | `chsrc_latest-1_amd64.deb` | **`.github/workflows/pkg-deb.yml:103,143`**——该文件名由 CI 的 stable-URL 重命名机制产生（`new_name="${old_name/${version}-1/latest-1}"`），**不是 README 里写的**；核验者别去 README 找 |
   | macOS | x64 / arm64 | `chsrc-{x64,aarch64}-macos` | `README.md` §安装·macOS |
   | macOS / Linux | — | `tool/installer.sh`（macOS 非 root 装至 `~/.local/bin`） | `README.md` §安装 |
   | **BSD** | — | **无预编译产物** | GitHub 无 BSD runner；`README.md` §安装·BSD 直接给的是源码编译命令 |

   Release 托管在 Gitee；GitHub 为上游主仓，Gitee 是 README 明示的可用替代源。

   **唯一真实平台缺口是 BSD。** 早前版本称「Linux AArch64 / ARMv7 / riscv64 是否发布未核实」是**错的**——README §安装·Linux 明确给出了这四个架构的下载 URL。`references/binary-resolution.md` 只需为 BSD 准备源码构建 / bootstrapper 兜底话术。

2. **预编译二进制必须锁版本 + 锁 SHA256（死结 C）。** §0.4 表中的 URL 指向 `pre` 这个**移动 tag**（`pkg-deb.yml:103` 另有 `latest` 重命名机制提供稳定下载 URL）。若同一轮 eval 的 baseline 组与 with-skill 组在不同时间点抓取，二进制可能已变，benchmark 对比被混淆。

   规定：每轮 eval 开始前抓取一次二进制，算出 SHA256，写入 `iterations/iteration-00N/eval_env.json`：

   ```json
   {
     "chsrc_version": "0.2.7.2",
     "source_url": "https://gitee.com/.../chsrc-x64-windows.exe",
     "sha256": "…",
     "fetched_at": "2026-10-06T…",
     "tag": "pre"
   }
   ```

   同一轮内两组配置**共用这一个文件**，中途不重新抓取。版本变更 → 该轮作废重跑。

3. 本机已装的 `chsrc 0.2.2`（WinGet）可作**降级备选**，仅作行为观察，不作正确性判据——版本落后仓库 0.2.7.2
4. **裸编译器调用是上游认可的合法构建路径，不是野路子。** chsrc 是**单文件编译项目**（`src/chsrc-main.c` 用 `#include` 汇聚全部框架与 recipe 的 `.c`，编译无中间目标文件），因此

   ```bash
   clang -Iinclude -Ilib src/chsrc-main.c -o chsrc
   ```

   就是**真实的构建方式**，不是「绕过构建系统」。`README.md` §安装·BSD 直接给出了这条命令，并紧接着说明「如果依赖还不存在，你会被死锁住，这才是 chsrc-bootstrap 起作用的时刻」。`justfile` / `Makefile` 是这条命令的封装。

   早前版本写的「不得教 AI 裸编译绕过构建系统」是**错的，已废止**。正确表述：Skill 优先给 `just` / `make`（上层封装，含 `-DXY_DEBUG` 等构建模式）；两者都不可用时，给出 `-Iinclude -Ilib` 的单文件编译命令，并在同一场景点明 bootstrapper 路线。`-Iinclude -Ilib` 是必需的，缺了会因找不到 `xy.h` 而编译失败。

### 0.5 Bootstrap 隔离环境的边界

| 项 | 规则 |
| :-- | :-- |
| 位置 | GitHub 隔离环境（专用验证仓库 / 工作流），**不在 `D:\chsrc` 本机** |
| 触发方式 | **不通过向 `D:\chsrc` 上游仓库推送来触发**——那会污染上游 CI |
| 可追溯 | 记录仓库 / commit SHA / 运行结果，便于复现 |
| 不可用时 | 标 `PLATFORM_UNAVAILABLE` 或 `BUILD_VALIDATION_SKIPPED`，**禁止虚报 PASS** |

具体载体（独立仓库 vs 上游 fork 的独立分支 vs 本地容器）见 §10 未决点 4。

### 0.6 判定规则

- 规格、成功标准、eval 设计中任何依赖**本机**构建产物的项，一律改写为静态核验或标记跳过
- 依赖 **Bootstrap 隔离环境**的项，标记为远程维度，在 `iterations/iteration-00N/result.json` 中与本地维度分开记录
- §7 记录的历史 `gcc` 编译实测属**已封存事实**，仅作环境参考，本阶段及以后不再复现
- 任何阶段若发现执行路径隐含**本机**构建，立即停止该阶段并记录 BLOCKER

---

## 1. 目的

开发一个名为 `chsrc` 的 Agent Skill，使 AI 助手在用户要求换源时，能正确地：

1. 识别该不该用 chsrc（trigger 判定）
2. 检测当前平台（Windows / Linux / macOS / BSD / Termux / 未知）
3. 按平台矩阵选择正确路径（预编译 / 源码编译 / bootstrapper）
4. 识别作用域（Project / User / System）并遵守安全约束
5. 在「无 chsrc + 无编译器 + 无预编译」的死锁场景下，走 bootstrapper 路线
6. 换源后用 `chsrc get <dish>` 复查当前源，不停留在临时源

## 2. 边界

**做：**

- Skill 的开发、eval、迭代、回归，全部在 `D:\chsrc\Skill` 内
- Skill 内容以「指引 AI 如何用 chsrc」为核心，包含平台检测矩阵、bootstrap 决策树、作用域策略、换源治理命令（`get`/`list`/`measure`/`set`/`reset`）与治理选项（`-dry`、`-scope=`）
- Windows 本地实机验证（PowerShell / CMD / MSYS2），**只读 CLI，不在本机构建**
- 本机 eval 运行环境使用**官方预编译 chsrc**（§0.4），取用而非编译
- Bootstrap 死锁场景在 **GitHub 隔离环境**验证（§0.2 / §0.5），构建是测试目的本身

**不做：**

- 修改 `D:\chsrc` 上游 C 代码（`src/`、`lib/`、`bootstrap/` 保持原样）
- **在本机构建上游 chsrc**（§0.3：禁 `just` / `make` / `gcc` / `clang` / 运行本地产物）
- 分发、打包、发布、复制到 `D:\ai-configs`
- 重新发明 chsrc 的 Build 逻辑——以 `justfile` / `Makefile` / `src/chsrc-main.c` 为唯一事实来源

**已知代价（用户已确认接受）：**

`Skill/` 位于上游仓库 `D:\chsrc` 工作树内，上游 `git status` 将持续显示未跟踪目录 `Skill/`。本项目不修改上游 `.gitignore`。

## 3. 协作门禁判定

| 项 | 判定 | 依据 |
| :-- | :-- | :-- |
| Team | 不可用 | Claude Code 无原生 team 运行时；`Agent` 工具的 `team_name` 参数已 deprecated |
| 并行编排 | 可用 | `Agent` 工具可并行 spawn；`SendMessage` 支持与已启动代理通信。主会话另有 `Workflow` 工具（spec 多维评审 / 35 case × 2 配置的 fan-out 走它），但**子代理侧不具备 `Workflow`**，只有 `Agent` + `SendMessage` |

按 Prompt.md §14，采用**并行编排路径**（Team 不可用 → 次选）。走 `Workflow` 的结论成立——Team 的 `team_name` 确已 deprecated——但依据要写准：**不是「Workflow 工具人人可用」，而是「并行 spawn 能力人人可用」**。子代理被要求 fan-out 时，用并行 `Agent` + `SendMessage` 达到同等并行度。

状态机：`START → Collaboration Gate → WORKFLOW → BRAINSTORM → SKILL DEVELOPMENT → SKILL CREATOR → DRAFT → CREATE EVALS → BASELINE/NEW RUN → REVIEW → FAILURE ANALYSIS → REWRITE → REGRESSION → IMPROVE / STABLE`

`NO_COLLABORATION_AVAILABLE → REPORT → STOP` 分支不触发。禁止伪造 Team 成员、伪造 Review、伪造 QA（§17）。

## 4. 目录结构

```text
D:\chsrc\Skill\
├── .ai\
│   ├── state\              current.md / tasks.md / blockers.md / collaboration-report.md
│   ├── decisions\          architecture.md / platform.md / bootstrap.md
│   └── verification\       local.md / ci.md
├── skill\chsrc\            ← 被迭代的 Skill 本体
│   ├── SKILL.md
│   ├── references\         platform-detection / binary-resolution / bootstrap /
│   │                       bootstrap-authoring / source-governance / scope-policy /
│   │                       ecosystem-detection / recovery / collaboration-workflow / ci-validation
│   ├── scripts\
│   ├── templates\           建空目录，填充时机待定（见 §10 未决点 3）
│   └── tests\
├── evals\
│   ├── evals.json          35 个 case
│   ├── files\
│   └── README.md
├── workspace\
│   ├── skill-snapshot\     旧版 Skill 快照（baseline）
│   ├── iteration-001\
│   ├── iteration-002\
│   └── iteration-003\
├── iterations\             每轮的 skill/ baseline/ with-skill/ analysis/ review/ result.json
├── tests\                  windows\ ci\ bootstrap\ safety\ collaboration\
├── scripts\                本项目自有工具脚本（eval 辅助、结果归档）
├── docs\
│   ├── design\             本规格
│   ├── brainstorming\      需求与方案发散记录
│   ├── iterations\         每轮的评审与复盘
│   └── decisions\          决策记录（与 .ai\decisions 互补：此处为人读）
├── .github\workflows\
└── README.md
```

`.tooling/anthropics-skills/` 不画在上图，因为它由 `.gitignore` 排除（见 §5 阶段 0 第 2 条），属 Development Tooling 而非 Skill 本体（Prompt.md §9）。

## 5. 具体动作

### 阶段 0 · 基础设施

1. `D:\chsrc\Skill` 下 `git init`，独立仓库（与上游仓库互不干扰）
2. 建立上述目录骨架**并写入 `.gitignore`**，排除运行时产物：

   ```gitignore
   .tooling/
   workspace/
   iterations/*/with-skill/
   iterations/*/baseline/
   evals/files/
   *.log
   ```

   保持跟踪：`iterations/*/result.json`、`analysis/`、`review/`、`evals/evals.json`、`iterations/*/skill/`
3. 建立**嵌套 git 仓库**（`Skill/.git`），锁住项目边界。这是唯一的真实防护机制：
   - 上游 `git add -A` **不吸收**嵌套仓库的内容（git 把有 `.git` 的目录视为独立仓库边界，只登记一个 gitlink 或直接跳过）
   - 上游 `git clean -fd` **跳过**嵌套仓库；`-ffd` 才会删除——而 `rules/filesystem.md` 对 `clean -fd` 有明确禁令
   - **注意**：「首次 commit 锁住骨架」这个说法是错的。commit 落在 `Skill/.git` 内，上游 `D:\chsrc` 看不见它，对上游操作没有任何约束力。起作用的**不是 commit，是 `.git` 目录本身的存在**
4. 在**上游本地** `D:\chsrc\.git\info\exclude` 追加一行 `Skill/`（**需用户授权**）。该文件是 git 的本地排除机制，不进版本控制、不影响任何克隆，因此**不违反 §2「不修改上游 `.gitignore`」**。效果是上游 `git status` 不再显示 `?? Skill/`，减少误操作面
5. 克隆 skill-creator 到 `.tooling/anthropics-skills`（Prompt.md §2 指定来源 `https://github.com/anthropics/skills`），记录 commit SHA 进 `.ai\decisions\architecture.md`
6. 记录 `.ai\state\collaboration-report.md`（Team 不可用 / Workflow 路径 / 状态 BLOCKED-CLEAR）
7. 记录 `.ai\decisions\architecture.md`：工作区归属、嵌套 git 为唯一防护机制、上游零改动、上游仓库禁止 `git add -A` / `git clean -fd`

### 阶段 1 · Skill 第一版（v0.1.0）

基于已勘察的上游事实撰写：

| 事实 | 来源 |
| :-- | :-- |
| CLI 命令集 = `get` / `set` / `reset` / `measure` / `list`，另有 `help` 与 `version` | `waiter.c:920` `waiter_handle_user_command()`；`chsrc-main.c:253,265` |
| **`dry` 是选项 `-dry`，不是命令**；`verify` 在 `src/` 全树无 CLI 绑定 | `chsrc-main.c:211`；`grep -rn verify src/` 仅命中 `chef.c:454` 内部函数 |
| 命令行选项必须写在菜品名之前 | `chsrc-main.c` 解析器每遇 `-` 即 `cli_arg_Dish_pos++`；CLAUDE.md |
| Scope = Project / User / System + `ImplementationDefinedScope`（默认，由 dish 自决） | `struct.h:122` |
| ScopeCapability = Unknown / Unable / Able_But_Not_Implemented / Able_And_Implemented | `struct.h:140` |
| Scope 选择 = `-scope=project\|user\|system\|default`；`-local` 已弃用并打 deprecation warning | `chsrc-main.c:167`（`-scope`）、`:157`（`-local`） |
| 其他选项 = `-ipv6` / `-en`(`-english`) / `-no-color`(`-no-colour`) / `-h`(`--help`) / `--version` | `chsrc-main.c:152,207,215,220,228` |
| dish 登记数 = **71** = `pl_*` 30 + `os_*` 31 + `wr_*` 10（dish 名为裸名，无前缀） | 按四个 `#define add` 分段统计：镜像站宏 `:122` → 计数区间 `:123-161` = 26；dish 段宏 `:166` / `:203` / `:237` → 计数区间 `:167-200` = 30、`:204-234` = 31、`:238-247` = 10。**三个 dish 区间均已排除宏定义行本身**（宏定义不计为登记项） |
| 镜像站（MirrorSite）**26** 个，登记于 `menu.c:125-156`，宏定义在 `menu.c:122`；与 dish 由 `#undef add`（`:162`/`:201`/`:235`/`:248`）分段，勿混计 | 逐段计数：`sed -n '123,161p' src/recipe/menu.c \| grep -c "add *("` = 26 |
| **计数陷阱**：`grep -c 'add *(' menu.c` = **101** 是匹配**行数**，`grep -o "add *(" \| wc -l` = **120** 是匹配**次数**。差额 **19** 来自 9 个一行多 `add(` 的行（`:127` 2、`:138` 4、`:139` 4、`:140` 3、`:143` 3、`:144` 2、`:145` 2、`:150` 3、`:156` 5，各超出 1）。**两个数都不是 26 或 71**，必须按四个 `#define add` 分段统计 | — |
| combo dish 需分菜品操作，不整体 set | `waiter.c:922-941` 遇 sub_dishes 打印套餐清单并递归 |
| 测速依赖外部 `curl`，缺失即报错退出 | `source.c` / `dish_select_fastest_source()` 前置 `chsrc_check_program_quietly_when_exist ("curl")` |
| bootstrap 死锁：预编译缺失 + 依赖缺失（编译器/make 也需换源才能装） | `bootstrap/README.md` |
| bootstrap 语言约定 = Windows PowerShell/CMD、Linux Bash/sh、macOS Zsh/Bash/sh、BSD sh | `bootstrap/README.md` §支持的语言 |
| **bootstrap 实现现状 = `bootstrap/` 目录内仅 `Termux.bash` 一个 bootstrapper**，其余平台为待实现 | `ls bootstrap/` = `README.md` + `Termux.bash` |
| **仓库另在 `tool/` 下有一键安装器（≠ bootstrapper）**：`installer.ps1`（Windows）/ `installer.sh`（Linux·macOS）。二者读取 CPU 架构（x86 / x64 / arm64），拼 `https://gitee.com/RubyMetric/chsrc/releases/download/pre/chsrc-<arch>-<platform>` 并 `Invoke-WebRequest` 下载**预编译二进制**。`installer.ps1` 支持 `-v` 指定版本；非 x86/x64/arm64 直接报 `Unsupported architecture` | `find tool -type f`；`tool/installer.ps1:151-199` |
| 构建目标 = `build-in-dev-mode` / `build-in-debug-mode` / `build-in-release-mode`；`just b`/`bd`/`br` 是 alias | `justfile:99-101` alias 段，`:110-121` 真实目标 |
| CI 只走 make，justfile 不含 CI 目标 | `justfile:28-30` 注释；`Makefile` 有 `bcir`；`.github/workflows/` 13 个文件（AArch64/ARMv7/riscv64/Windows Arm64/x64/macOS） |

SKILL.md 保持短（目标 < 300 行），细节全部下沉到 `references/`。

### 阶段 2 · Eval 集（35 case）

覆盖面（Prompt.md §21）：

| 维度 | case 数 | 覆盖的 CLI 形式 / 要点 |
| :-- | --: | :-- |
| Trigger | 6 | 4 正例（"把 pnpm 换成国内镜像"）+ 2 负例（手改配置文件、非镜像类安装） |
| Platform Detection | 6 | Windows / Linux / macOS / BSD / Termux / Unknown（Prompt.md §34 六项） |
| Install · Prebuilt · Build | 4 | 预编译可用 / 缺失走编译 / `just` vs `make` 选择 |
| Bootstrap · 入口 | 4 | **纯静态断言，本机跑**（§0.1.1）：Termux 走真实实现断言（`Termux.bash` 存在）；Windows/Linux/macOS/Unknown 断言「识别出无现成 bootstrapper → 给出 authoring 路径」 |
| Bootstrap · 死锁复现 | 1 | **需构建，只在 GitHub 隔离环境**（§0.2）。载体未定前标 `BUILD_VALIDATION_SKIPPED`。断言目标已降维——见 §8「死锁前提在载体上不可复现」 |
| Bootstrap Handoff | 2 | **出口分支**（交接完整性，Prompt.md §39）：预置「bootstrapper 已执行完毕」的场景 fixture，断言 AI 的回答包含 `chsrc --version` 验证 + 交接给 chsrc 的源治理步骤，且未停留在临时 Source |
| Source Governance | 4 | `get` / `list` / `measure` / `set` / `reset`（**无 `dry`/`verify` 命令**，`-dry` 归 Safety 维度） |
| Scope & Safety | 4 | `-scope=` 三作用域、系统级 pip 源、`-dry` 预览 |
| Container / Docker | 2 | `wr_docker` dish（`menu.c:241`）；容器内无 chsrc 时的路径选择 |
| Recovery | 2 | 换源失败回退、`reset` 复原 |

合计 35 case。schema：`{ id, prompt, expected_output, files, expectations }`

### 阶段 3 · 执行（完整集 · 多轮迭代）

用户已确认「完整集·多轮迭代」。单轮成本 = 35 case × 2 配置 = 70 个子代理运行。

**baseline 三态定义**（Prompt.md §26 指出第一轮不存在 old skill）：

| 轮次 | baseline | 衡量的是 |
| :-- | :-- | :-- |
| 第 1 轮 | **无 skill**（不注入 `skill/`，产出存 `baseline/none/`） | Skill 带来的绝对增益 |
| 第 N≥2 轮 | `iterations/iteration-00(N-1)/skill/`（上一轮 with-skill 实测版） | 本轮改动的增量 |

`workspace/skill-snapshot/` 保存「上一轮 with-skill 实测版」，仅第 2 轮起生效。

每轮流程（skill-creator 规范）：

1. 按上表确定 baseline，准备 `with-skill` 与 `baseline` 两组输入目录
2. 同一轮内同时 spawn with-skill 与 baseline 两组子代理
3. 捕获 `timing.json`（`total_tokens` / `duration_ms`）
4. grader 按 `expectations` 判定 → `grading.json`（字段固定为 `text` / `passed` / `evidence`）
5. `aggregate_benchmark` → `benchmark.json` / `benchmark.md`
6. `generate_review.py` 出 review 界面，人工评审
7. 读 `feedback.json` → 失败分析（区分 Skill 缺陷 / Eval 缺陷 / 环境缺陷 / 模型缺陷）→ 改 Skill。分析同时写入 `iterations/iteration-00N/analysis/` 与 `.ai\decisions\`
8. 回归：历史 case 全保留，只增不减（§25）

脚本调用统一走 `.tooling/anthropics-skills/skills/skill-creator/` 下的相对路径；脚本缺失即报 BLOCKER 并停止该轮，不静默跳过。

轮次规划：最多 3 轮。每轮结束产出 `iterations/iteration-00N/result.json`。

**停止条件（Prompt.md §49 + §50）：**

Prompt.md 要求 8 项全过。本项目有 1 项受构建约束影响，处理方式写成显式豁免条款，避免「合取里藏着一个永假项」这种无法收敛的结构：

```text
必过 7 项（合取，全部 PASS 才算）：
  Critical Eval PASS
  Safety PASS
  Bootstrap PASS
  Cross-platform PASS
  Regression PASS
  Human Review PASS
  Windows Local PASS

第 8 项  Available CI Platforms —— 不参与合取
  ├─ 载体（§10 未决点 4）已就绪 → 实际运行，记 PASS / FAIL
  └─ 载体未定               → 记 BUILD_VALIDATION_SKIPPED（§0 授权豁免）
```

`Available CI Platforms` 一旦记为 FAIL，视为阻塞项，与前 7 项同级处理；只有 `SKIPPED` 才是豁免。载体就绪后该豁免自动失效，8 项恢复为完整合取。

## 6. 成功标准

| # | 标准 | 验证方式 |
| :-- | :-- | :-- |
| 1 | Skill 目录结构完整，SKILL.md frontmatter 合法 | 文件存在性检查 |
| 2 | SKILL.md 内容与上游代码一致，无捏造命令 | 逐条比对**源码**：`waiter.c` / `struct.h` / `chsrc-main.c` / `menu.c` / `justfile` / `Makefile`。**`CLAUDE.md` 仅作参考，不作判据**——它自述「架构速览仍引用拆分前的旧文件名（`core.c` 等）」已过时，拿它当事实源会形成循环论证，让 Skill 继承的错误通过校验 |
| 3 | 35 个 eval case 全部可执行 | `evals/evals.json` schema 校验 |
| 4 | 每轮产出 grading / benchmark / analysis / review | 目录完整性检查 |
| 5 | 回归 case 数量单调不减 | 各轮 evals 数量对比 |
| 6 | **静态事实一致**：SKILL.md 与 references/ 中每条命令、选项、dish、scope 语义均可回溯到上游源码行号 | grep 核验 + 逐条比对 `waiter.c` / `struct.h` / `chsrc-main.c` / `menu.c`（替代原「可构建」标准，见 §0） |
| 7 | **Skill 行为**在 Windows 实机验证（governance + bootstrap 决策） | `tests/windows/` 下 PowerShell / CMD / MSYS2 三套脚本各跑一遍，**使用官方预编译 chsrc**。脚本内**所有 chsrc 写操作一律带 `-dry`**，只做只读 CLI 调用（`--version` / `list` / `get <dish>` / `help`）；探测编译器用 `command -v` 只查存在性，**不调用**；不构建 |
| 8 | **Bootstrap 死锁场景**在 GitHub 隔离环境验证 | 远程执行：`chsrc-bootstrap → 安装编译依赖 → make/just → 构建 chsrc`（§0.2）。本机零构建。载体待定（§10 未决点 4），未就绪前标 `BUILD_VALIDATION_SKIPPED`，禁止虚报 PASS |
| 9 | `D:\ai-configs` 全程零改动 | `git status` 前后对比 |
| 10 | 上游 `src/` `lib/` `bootstrap/` 零改动 | 上游 `git status` 仅多出 `Skill/` |
| 11 | eval 执行环境隔离（Safety case 不污染真实配置） | **Windows 子代理禁止任何非 `-dry` 写操作**（见下方说明）；Linux/macOS 子代理可用临时 `HOME` 辅助 |
| 12 | eval 运行环境的 chsrc 来源可追溯 | 记录预编译二进制的版本 + 来源 URL + SHA256（§0.4），不落在本机构建 |
| 13 | **Bootstrap Handoff 契约**（Prompt.md §39 的 NO_BUILD 等价覆盖） | 静态文本契约：(a) `SKILL.md` / `references/bootstrap.md` 含显式 `chsrc --version` 验证步骤；(b) 含「bootstrapper 只切临时源，必须交接给 chsrc」的显式声明。行为面：2 个 Bootstrap Handoff case（预置已执行 fixture）断言 AI 输出交接步骤而非停留在临时源。**`BUILD_VALIDATION_SKIPPED` 豁免的只是真实编译，handoff 契约本身必须 PASS** |
| 14 | Skill 中的源码断言可机器复核 | `scripts/verify-facts.py` 退出码 0 且断言覆盖率 100%（见 §11） |

### 6.1 为什么 `-dry` 是唯一可靠隔离（`HOME` 在 Windows 上无效）

上一版规格写「所有写操作用 `-dry`，或临时 `HOME` 内执行」。后半句在 Windows 上是**假的安全**：

| 作用域 | Linux/macOS 落点 | Windows 落点 | `HOME` 覆盖有效？ |
| :-- | :-- | :-- | :-- |
| user scope | `~/.npmrc` 等 | `%APPDATA%` / `%LOCALAPPDATA%` | **否** |
| system scope | `/etc/*` | `C:\ProgramData` + 注册表 | **否** |

本机是 Windows，且 Windows 是主要验证平台，`HOME` 隔离在这里**一点用都没有**。而 chsrc 的系统级作用域会写 `C:\ProgramData` 与注册表——这是最难回收的一类污染。

`-dry` 是 chsrc 的**全局闸门**，已核实覆盖全部写路径：

| 守卫位置 | 覆盖的原语 |
| :-- | :-- |
| `OS.c:324` | `chsrc_run()` — `system(cmd)` 命令执行 |
| `OS.c:382` | `chsrc_run_directly()` — 直连执行 |
| `OS.c:750` / `805` / `836` / `875` | 含 `chsrc_overwrite_file`（`fopen` 前）、`chsrc_append_to_file`（`fopen` 前） |
| `source.c:355` / `370` | 跳过测速（`pre` tag） |

入口处 `chsrc-main.c:242` 打印 Dry Run 提示。**结论：所有写操作一律 `-dry`，无例外；`HOME` 隔离只在 Linux/macOS 子代理上作为额外一层。**

## 7. 本机工具链现状（2026-10-06 实测 · 已封存）

> **封存声明：** 下表的编译器可用性来自本项目启动前的一次环境勘察。其中标注为「实测」的编译结果**属历史记录，本阶段及以后不在本机复现**——§0.3 已将其封存。本节的现实用途是：让 Skill 知道「用户机器上装了什么、缺什么」，从而在指导换源/安装时给出正确判断。

| 工具 | 状态 | 证据 |
| :-- | :-- | :-- |
| `chsrc` | **已安装 0.2.2** | `WinGet\Packages\RubyMetric.chsrc_...`；落后仓库 0.2.7.2 |
| `gcc` | 存在（**禁止调用**） | `gcc.exe (Rev8, Built by MSYS2 project) 15.2.0` |
| `perl` | 存在 | `perl 5.38.2 built for x86_64-msys-thread-multi` |
| `python` | 存在 | `Python 3.13.9` |
| `just` | **缺失** | `command -v just` 无结果；`~/.cargo/bin`、`ProgramData/chocolatey/bin` 均无 |
| `make` | **缺失** | `command -v make` 无结果；`/mingw64/bin`、`/usr/bin` 均无 |
| `clang` | **缺失** | — |
| 上游构建产物 | **无** | `D:\chsrc` 下无 `chsrc` / `chsrc-debug` / `chsrc-release` |

**已封存事实（本机不再复现）：** 早期勘察中执行过一次 `gcc -Iinclude -Ilib -DXY_DEBUG src/chsrc-main.c`，编译成功，产物报告 `chsrc v0.2.7.2`。该记录仅用于确认 `-Iinclude -Ilib` 是必需包含路径，**不构成任何执行许可**。（Bootstrap 隔离环境中的构建不受此封存约束，见 §0.2。）

**对 Skill 内容的影响（重要）：**

chsrc 上游文档大量使用 `just bd` / `make bd`。Skill 涉及「安装 / 构建 chsrc」的指引必须写成**条件式**：

```text
检测构建工具（just / make / cc）
  ├─ 存在 → 给出对应命令（构建是用户自己机器上的事，本 Skill 只给出命令）
  └─ 缺失 → 明确告知「当前环境无构建工具」，转向官方预编译安装路径
```

三点边界要区分清楚，「谁构建」和「是否执行」是两个独立维度：

| 行为 | 允许？ | 说明 |
| :-- | :-- | :-- |
| Skill 文本里**写着**「装依赖、跑 `make`」 | **允许** | Skill 的本职工作就是告诉 AI 该怎么做，用户机器上该不该构建由用户决定 |
| 本项目开发方在**本机执行**构建 | **禁止** | §0.3，开发机与本机 eval 环境不编译 |
| eval 子代理**执行**构建 | **禁止** | §0.1.3，子代理只输出命令文本供 grader 判定 |

第三行是第二轮评审补上的：前两行只划清了「谁构建」，没划「是否执行」。eval 场景下两者会重合——Skill 文本会让子代理想照做，这是死结 A 的制度成因。

「检测构建工具」这一分支的收敛也要分三种情况（2026-10-06 更正）：

| 情形 | 给出什么 |
| :-- | :-- |
| `just` / `make` 存在 | 优先给 `just b` / `make b`（上层封装，含 `-DXY_DEBUG` 等模式） |
| 两者都缺，但有编译器 | 给单文件编译 `clang -Iinclude -Ilib src/chsrc-main.c -o chsrc`。**这是上游认可的真实构建方式**（chsrc 是单文件编译项目），`README.md` §安装·BSD 就是这么写的 |
| 编译器也缺 | 走 bootstrapper，见 `references/bootstrap.md` |

早前版本写的「Skill 不得把裸编译当兜底推荐——绕过构建系统不是合法路径」是**错的，已废止**。上游事实是：单文件编译就是构建本身，`justfile` / `Makefile` 是它的封装。`references/platform-detection.md` 记录工具探测方法（只探测是否存在，不执行）。

---

## 8. 风险与应对

| 风险 | 影响 | 应对 |
| :-- | :-- | :-- |
| 180-260 个子代理运行开销大 | 耗时数小时 | 每轮设检查点，用户可随时叫停；轮次上限 3 |
| **eval 子代理直接跑在开发机，Safety 维度涉及 `chsrc set` 写操作** | **污染用户真实 pip/npm 配置，且失败样本无法复现** | **Safety case 一律 `-dry`**（§6.1 已核实为全局闸门，覆盖命令执行与文件写入两条路径）；**Windows 子代理禁止任何非 `-dry` 写操作**；Linux/macOS 可叠加临时 `HOME`。见 §6 标准 #11 |
| **上游仓库里的未跟踪 `Skill/` 目录** | **上游任何人执行 `git add -A` 或 `git clean -fd` 会带走整个项目**（`rules/filesystem.md` 对 `clean -fd` 有明确禁令） | 阶段 0 第 3 条建立**嵌套 git 仓库**（真正的防护机制），第 4 条在 `.git/info/exclude` 加 `Skill/`（需用户授权）；`.ai\decisions\architecture.md` 写明上游仓库禁止 `git add -A` / `git clean -fd`。**注意防护来自 `.git` 目录的存在，不是 commit** |
| **Windows 上 `HOME` 隔离无效** | system scope 写 `C:\ProgramData` + 注册表，user scope 走 `%APPDATA%`，均不受 `HOME` 覆盖 | §6.1：所有写操作一律 `-dry`（已核实为全局闸门，6 处守卫）；Windows 子代理禁止任何非 `-dry` 写操作 |
| **静态事实标准无执行机制** | §6 #6/#2 会退化成「我 grep 过了」，永远无法判失败 | §11 定 `scripts/verify-facts.py`：抽取断言 → 回读源码行 → 覆盖率 100% + 退出码 0 为 PASS |
| **`.tooling/anthropics-skills` 依赖网络与上游仓库结构** | **脚本改名或仓库结构变动 → 阶段 3 中途失败** | **clone 后记录 commit SHA；脚本走相对路径调用，缺失即报 BLOCKER 而非跳过** |
| Skill 内容与上游代码漂移 | eval 失效 | 每次迭代前复核上游事实表 |
| eval case 本身设计有误 | 指标失真 | 失败分析强制区分 Eval 缺陷 / Skill 缺陷 |
| Bootstrap 跨平台无法在本机验证 | 该维度依赖 GitHub 隔离环境（§0.5） | 载体就绪前标 `PLATFORM_UNAVAILABLE` 或 `BUILD_VALIDATION_SKIPPED`，禁止虚报 PASS |
| 上游仓库出现未跟踪目录 | 用户已确认接受 | 不修改上游 `.gitignore` |
| **违反本机不构建约束（§0.3）** | **用户明确禁止的越界行为** | 阶段 0 写入 `.ai\state\current.md` 与 `.ai\decisions\architecture.md`；任何隐含本机构建的路径标 `BUILD_VALIDATION_SKIPPED`，不「顺手跑一下」 |
| **Bootstrap Handoff 无法实机验证**（Prompt.md §39 的行为闭环） | 静态 grep 只能证明 Skill 写了「后续交给 chsrc」，证明不了 AI 真交接了 | 转成「可静态验证的文本契约 + 预置 fixture 的场景推断」两个层次（§6 标准 #13）。`SKIPPED` 只豁免真实编译，handoff 契约必须 PASS |
| **Bootstrap 隔离环境载体未定**（§10 未决点 4） | §6 标准 #8 退化为 `SKIPPED`，「死锁复现」case 拿不到实测结果 | 需用户在 (a) 独立验证仓库 / 其他方案间定夺（建仓是对用户 GitHub 账号的外部写入，须本人授权）。未定前**只有「死锁复现」1 个 case** 标 `BUILD_VALIDATION_SKIPPED`，Bootstrap 入口与 Handoff 维度照跑（§0.1.1） |
| **死锁前提在载体上不可复现** | §0.2 假定「没有编译器时怎么办」，但 GitHub 美欧 runner 网络畅通、`apt`/`curl` 齐备，`apt update` 直连官方源就成功，「要装编译器就得先换源」这个前提**不成立** | 断言目标显式降维为「bootstrapper 能在无编译器环境下引导装上编译器并完成构建」（可行动的部分）；「必须先换源才能装编译器」记为**已知局限**，禁止把 happy path 测成死锁 PASS |
| **预编译二进制版本漂移** | §0.4 的 URL 指向移动 tag `pre`，baseline 组与 with-skill 组若在不同时间点抓取，benchmark 对比被混淆 | 每轮抓一次 + 记 SHA256 落 `iterations/iteration-00N/eval_env.json`，同轮两组共用（§0.4 第 2 条）；版本变更该轮作废重跑 |
| **预编译下载失败连坐多个维度** | 网络不可达 / Gitee 故障时，Trigger、Platform Detection 等纯推理维度被无辜打掉 | 按 §0.1.1 拆分维度依赖：预编译只服务需要可执行 chsrc 的 5 个维度，其余 5 个不受影响 |
| **预编译 chsrc 与仓库版本不一致** | eval 观察到的行为可能与源码事实冲突，误判 Skill 缺陷 | 记录版本 + 来源 + SHA256（§6 标准 #12）；版本落后时该 case 结论标注版本差异，不计入 Skill 失败 |
| **「教用户构建」与「本机不构建」被混淆** | Skill 被写成不敢提构建，或反过来开发机被编译 | §7 明确划线：Skill 教构建 = 允许，本项目代为构建 = 禁止；评审时按此判定 |

## 9. 应急

- 单轮 eval 中某个 case 子代理崩溃 → 重跑该 case，不重跑整轮
- 上游代码变动 → 停止迭代，重核事实表，规格升版
- 连续两轮无指标提升 → 停机，输出当前最佳版与未解决点，交用户决定
- 触发 `NO_COLLABORATION_AVAILABLE` → 写 `.ai\state\collaboration-report.md` 后 STOP
- **任何阶段发现必须在本机构建 chsrc 才能推进** → 该维度标 `BUILD_VALIDATION_SKIPPED` 记入 `iterations/iteration-00N/result.json`，跳过该维度，**不得为"验证一下"而编译**

## 10. 未决点

1. **迭代轮次上限**：规格写 3 轮。达到 `LOCAL_STABLE` 条件可提前停。
2. **是否需要 `scripts/`**：上游 `test/cli.pl` 已是可执行验证器，Skill 是否需自带探测脚本待第一版写出后判断。
3. **`templates/` 用途**：Prompt.md §31/§54 要求存在。阶段 0 建空目录，内容与填充时机待第一版 Skill 写出后判断（YAGNI 下不预填）。
4. **Bootstrap 隔离环境的载体** —— 需用户定夺。评审已逐一评估：

   | 候选 | 结论 | 理由 |
   | :-- | :-- | :-- |
   | **(a) `Swcmb/` 下独立验证仓库 + 工作流** | **推荐** | 工作流用 `actions/checkout` 的 `repository: RubyMetric/chsrc` + `ref: <SHA>` 拉源码，**零上游污染**（不向上游产生任何 ref）；锁 SHA 即锁可复现性，比拉最新 `dev` 强得多；工作流在自己仓库里可自由迭代。成本：Linux 容器分钟便宜，macOS runner 贵一个量级 |
   | (b) 上游仓库独立分支 + 远程 API | **淘汰** | 向 `RubyMetric/chsrc` 建 ref 需要上游 push 权限（用户没有）；用 fork 的话与 (a) 完全等价却多一层间接。且原文「用远程 API 在独立 ref 上跑」**与 §0.5 自己写的禁令直接矛盾**——远程 API 向上游建 ref 就是污染上游 |
   | (c) 本地容器 + 远程镜像预热 | **淘汰** | 容器内跑 `make build-in-release-mode` 就是在本机构建 chsrc，与 §0.3「本机禁止 `make`（任意目标）」正面冲突。要合法使用须把 §0.3 改成「非容器环境禁构建」，分域从二维膨胀为三维（环境 × 是否容器），复杂度陡增，省下的仅是 GitHub 分钟数 |

   **两件需要用户本人拍板的事：**

   - 建仓是对用户 GitHub 账号的**外部写入**，须本人授权，agent 不自行创建
   - 许可：克隆并构建 GPL-3.0-or-later 的 chsrc 属「使用」；只要不上传构建产物当 release，就无分发义务

   **载体换不解决的限制：** GitHub 官方 runner **没有 BSD**，BSD bootstrap case 无论如何只能 `PLATFORM_UNAVAILABLE`——平台可得性问题，与载体选择无关。

   载体定下来后填入本节，并同步 `.ai\decisions\bootstrap.md`。
---

## 11. 静态事实核验机制

§6 标准 #6 与 #2 都是「与上游源码一致」。这类标准最常见的失效方式是**退化成「我 grep 过了」这种不可复核的声明**——它没有执行者、没有产物、没有阈值，因此永远无法判定失败。本节把它固定成一个脚本。

### 11.1 断言格式

`SKILL.md` 与 `references/*.md` 中每条可回溯到上游源码的断言，写成机器可读形式，集中在 `evals/facts.json`：

```json
{
  "assertions": [
    {
      "id": "cli-command-set",
      "claim": "CLI 命令集 = get / set / reset / measure / list，另有 help 与 version",
      "source": "src/framework/waiter.c",
      "line": 920,
      "must_contain": ["get", "set", "reset", "measure", "list", "help", "version"]
    },
    {
      "id": "dry-is-option-not-command",
      "claim": "dry 是选项 -dry，不是命令；verify 无 CLI 绑定",
      "source": "src/chsrc-main.c",
      "line": 211,
      "must_not_contain": ["\"verify\""]
    }
  ]
}
```

规则：

- `source` + `line` 指向 `D:\chsrc`（**只读**，脚本不得写上游）
- `must_contain` / `must_not_contain` 是该行需匹配 / 不得匹配的 token
- 每条断言须在 SKILL.md 或某个 reference 中**被正文引用**（脚本反查），避免写了断言但正文没写

### 11.2 脚本行为

`scripts/verify-facts.py`：

1. 逐条读 `source:line`，校验 `must_contain` 全部命中、`must_not_contain` 全部未命中
2. 反查每条断言的 `claim` 是否真的出现在 SKILL.md / references/ 中（覆盖率检查）
3. 上游文件若有行号漂移，报错并提示重新定位，**不猜测**、不模糊匹配
4. 输出 `iterations/iteration-00N/fact-check.json`（逐条 pass/fail + 证据行）
5. 退出码：全过 = 0，任一失败 = 1

### 11.3 与评测的关系

- §6 标准 #6 与 #2 的判定方式改为「该脚本退出码 0 且覆盖率 100%」
- 上游代码变动时，脚本先失败 → 复核事实表 → 更新 `facts.json` 行号 → 再跑评测。**这取代了「每次迭代前人工复核事实表」**这条不可复核的约定
- 脚本本身属本项目工具，不属 Skill 本体，不打包进最终 Skill
