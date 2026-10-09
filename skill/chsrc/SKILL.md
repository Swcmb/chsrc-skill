---
name: chsrc
description: >-
  This skill should be used whenever the user asks to change a package manager's
  mirror/源 (换源、换镜像、国内镜像、加速源) for pip, npm, pnpm, yarn, cargo, rustup,
  go, gem, composer, conda, apt, yum, pacman, winget, brew, docker, nix, emacs,
  tex, anaconda, homebrew, flatpak, or the OS distro's own package manager —
  or asks to install chsrc itself, or to inspect the current source
  (查看当前源/当前镜像/测速). Also use it for "chsrc get/set/reset/list/measure",
  "换回官方源", and for diagnosing why a mirror is unreachable. Do NOT use it when
  the user only wants to edit a config file by hand, or for a non-mirror task such as
  GitHub SSH, proxy/VPN setup, or building software from source.
---

# chsrc 换源指南

chsrc 是一个用 C11 写的跨平台换源工具（Linux / Windows / macOS / BSD / Termux，多架构）。
本 Skill 提供换源决策所需的**上游实证知识**——命令集、作用域语义、平台矩阵、bootstrapper
现状——而不是即兴推测。

**核心工作流：换源 → 复查。** 换完必须用 `chsrc get <dish>` 验证当前源，不停留在临时源。

---

## 一、先判断该不该用 chsrc

判定有顺序。**先分类用户的问题，再考虑工具**——顺序反了会把「项目依赖装不上」误判成
「源太慢」，继而把整个回答变成测速报告。

### 第 1 步 · 用户问的是公共源镜像吗

只有当用户想改变「这个工具下载依赖时走哪个公共镜像站」时，才进入 chsrc 流程：

- 「把 pnpm 换成国内镜像」「pip 换清华源」「npm 走淘宝源」「conda 加速」
- 「换回官方源」「重置 pip 源」
- 「我现在用的什么源」「哪个镜像快」
- 「装一下 chsrc」「chsrc 报错」

### 第 2 步 · 不是的话，别碰 chsrc

| 用户说的是 | 正确做法 |
| :-- | :-- |
| **某个项目依赖装不上**（「这个 Python 项目依赖装不上」） | 先看该项目的 `requirements.txt` / `pyproject.toml` / `package.json`，判断是缺依赖、版本冲突、解释器错配还是锁文件问题。项目内的依赖问题由项目自己的依赖清单定义 |
| 让用户手改 `~/.npmrc` / `pip.ini` / `.condarc` | 除非用户明确要求手改，否则走 chsrc |
| GitHub SSH、代理/VPN、DNS 调优 | 与换源无关 |
| 源码编译某个项目 | 与换源是两件事 |
| 私有源 / 公司内网源 | chsrc 管公开镜像，私有源走各自的配置文件 |

**「装不上」和「太慢」是两回事。** 装不上通常是依赖清单或环境问题，太慢才轮到换源。
用户说「装不上」时，先诊断装不上的原因；只有在证据指向公共镜像站不可达或过慢时，
才把换源作为**排查项之一**提出，且明确它是可选项而非默认动作。

排查项的前提是**用户的问题已经落在某个依赖工具上**（例如 pip 装包慢）。
用户的问题与包管理无关时（Git SSH 推送失败、代理连不上、DNS 解析异常、编译报错、
CI 跑不起来），**整段换源内容不输出**——不要在回答末尾以「顺便说一句」「另外」之类的
方式追加 chsrc 换源建议，即使只是一句提点。此时提 chsrc 与用户的问题无关，属于噪声。

用户描述模糊时，先问一句，比猜准。

---

## 二、执行流程

```text
用户要换源
   ↓
[1] 确认 chsrc 是否可用 → 没有则走「三、获取 chsrc」
   ↓
[2] 检测平台与架构 → 见 references/platform-detection.md
   ↓
[3] 识别目标 dish（pip? npm? apt? winget?）→ 见 references/ecosystem-detection.md
   ↓
[4] 选作用域（Project / User / System）→ 见 references/scope-policy.md
   ↓
[5] 先预览 -dry，再执行 set，最后 get 复查 → 见 references/source-governance.md
```

**顺序不能颠倒**：作用域在预览之前就要定，否则 `-dry` 预览的 scope 和实际执行的 scope 不是一回事。
复查（`get`）永远在最后一步。

---

## 三、获取 chsrc

### 已有 chsrc

```bash
chsrc --version      # 确认可执行
```

### 没有 chsrc

按优先级：

| 顺序 | 方式 | 适用 |
| :-- | :-- | :-- |
| 1 | 包管理器（`winget install RubyMetric.chsrc` / `scoop install chsrc`） | Windows |
| 2 | 官方预编译二进制 | 见下表 |
| 3 | 源码构建（`just` / `make`） | 前两者都不可用 |
| 4 | bootstrapper | 编译器本身也需要换源才能装时（见 `references/bootstrap.md`） |

官方预编译二进制：

| 平台 | 架构 | 来源 |
| :-- | :-- | :-- |
| Windows | x64/x86/arm64 | `https://gitee.com/RubyMetric/chsrc/releases/download/pre/chsrc-{x64,x86,arm64}-windows.exe` |
| Linux | x64/aarch64/riscv64/armv7 | `chsrc-{x64,aarch64,riscv64,armv7}-linux` |
| macOS | x64/arm64 | `chsrc-{x64,aarch64}-macos` |
| **BSD** | — | **无预编译产物**，见下 |

Release 托管在 Gitee（GitHub 不可达时的替代源）。上游 `tool/installer.ps1` 会自动探测
x86/x64/arm64 并拼接 URL，支持 `-Version` 指定版本，可以直接用它。

**只有 BSD 没有预编译产物**（GitHub 无 BSD runner）。BSD 上走源码构建；依赖本身也缺时走
bootstrapper。

---

## 四、命令参考

chsrc 的动作只有五个，另有 `help` 与 `version`。每个动作都有短别名——**用全名最安全**，
短别名里 `c` 和 `m` 都指测速，`r` 既是 reset 也不是别的：

| 动作 | 全名与全部别名 |
| :-- | :-- |
| 查看当前源 | `get` · `g` |
| 换源 | `set` · `s` |
| 恢复默认源 | `reset` · `rest` · `r` |
| 测速 | `measure` · `mea` · `m` · `cesu` · `ce` · `c` |
| 列出信息 | `list` · `ls` |
| 帮助 | `help` · `h` · `-h` · `-help` |
| 版本 | `version` · `--version` |

```bash
chsrc get <dish>              # 查看当前源（只读）
chsrc set <dish> [源]         # 换源（写操作）。第二个位置参数可指定镜像站
chsrc reset <dish>            # 恢复默认源（写操作）
chsrc measure <dish>          # 测速各候选源（只读）
chsrc list <dish>             # 列出该 dish 的信息；套餐 dish 会先展开子菜品
chsrc ls                      # 不带参数时列出全部可用镜像站及其 code
chsrc help
chsrc version
```

### 指定镜像站

`set` 的第二个位置参数接受**镜像站 code** 或**完整 URL**。不给时 chsrc 会测速后自动挑最快的：

```bash
chsrc set pip tuna                              # 指定清华 TUNA
chsrc set pip https://mirrors.example.com/       # 或直接给自定义源 URL
chsrc set pip                                   # 不给 → 测速后自动选
```

镜像站 code 用 `chsrc ls` 查（`tuna` / `ustc` / `ali` / `tencent` / `bfsu` / `zju` / …）。
用户说「换到清华」时直接用 `chsrc set pip tuna`，比让它测速更快也更可控。

**没有 `dry` 命令**，`dry` 是选项 `-dry`。**没有 `verify` 命令**——换完源用 `get` 复查。

### 选项

**选项必须写在 dish 名之前。** chsrc 的解析器每遇到一个 `-` 就把菜品参数位后移一位，
所以 `chsrc get pip -no-color` 会把 `-no-color` 当成菜品名，报「暂不支持的换源菜品」。
正确写法：

```bash
chsrc get -no-color pip        # ✓
chsrc get pip -no-color        # ✗
```

常用选项：

| 选项 | 作用 |
| :-- | :-- |
| `-dry` | **预览**。模拟换源、跳过测速，只打印不执行。任何写操作前先加它确认 |
| `-scope=system` / `-scope system` | 指定作用域。空格与等号两种写法都支持 |
| `-local` | **已弃用**，会打 deprecation warning。改用 `-scope=project` |
| `-ipv6` | 测速走 IPv6 |
| `-en` / `-english` | 英文输出 |
| `-no-color` / `-no-colour` | 关闭颜色 |

### 常见 dish

| 生态 | dish |
| :-- | :-- |
| Python | `pip` `poetry` `pdm` `uv` `rye` `py_group`（套餐） `uv_pypi_index` `uv_python_build` |
| Node | `npm` `pnpm` `yarn` `bun` `nvm` `js_group`（套餐） |
| 其他语言 | `ruby` `php` `perl` `lua` `go` `rust_cargo` `rust_rustup` `java` `clojure` `dart` `dart_flutter` `nuget` `haskell` `ocaml` `r` `julia` |
| 发行版 | `ubuntu` `debian` `fedora` `arch` `archlinuxcn` `manjaro` `alpine` `opensuse` `kali` `msys2` `termux` 等 31 个 |
| 通用软件 | `winget` `homebrew` `docker` `flatpak` `nix` `guix` `emacs` `tex` `anaconda` `cocoapods` |

不确定某个东西对应哪个 dish 时，先跑 `chsrc ls` 看镜像站清单、`chsrc list <dish>` 看单个 dish 的
详情（含别名、维护者、能力声明）。dish 名不区分语言：用户说「python」要映射到 `pip`，
说「pip 换源」则 dish 就是 `pip`。见 `references/ecosystem-detection.md`。

### 套餐 dish

`py_group`、`js_group`、`uv` 是**套餐**（combo dish）。对它们执行 `get`/`set`/`reset` 时，
chsrc 会先列出子菜品构成，再**逐个**操作。不能当成单个对象整体替换。

### 当 chsrc 版本旧、某个 dish 不会自动落盘

部分旧版本 chsrc 的 `set <dish>` 只**打印**要写入的内容，不替你写文件（例如 cargo 自动
换源是 2025-09-13 之后才进上游）。遇到这种情况，处置顺序是：

```text
1. 查版本：chsrc --version
2. 先升级 chsrc —— 装最新版，让它自动完成换源
   winget upgrade RubyMetric.chsrc   /   scoop update chsrc
3. 升级后重跑 chsrc set <dish> <code>，由它自动落盘
4. 只有在无法升级（无管理员权限 / 离线 / 平台无包管理器）时，
   才手动把 chsrc 打印出来的内容写进配置文件——这是兜底，不是首选
```

升级是首选，因为手动改配置文件会绕过 chsrc 的备份机制，也容易覆盖掉配置文件里
chsrc 不管理的其他段落（例如 cargo 的 `[net] git-fetch-with-cli`）。

不要跳过升级直接教用户手改配置文件。

---

## 五、作用域与安全

三种真实作用域：

| 作用域 | 含义 |
| :-- | :-- |
| `ProjectScope` | 仅当前项目目录 |
| `UserScope` | 当前用户全局 |
| `SystemScope` | 全系统。Linux 写 `/etc`；Windows 上由各 dish 自行决定，多数委托给厂商命令行（如 `winget source add`） |

外加 `ImplementationDefinedScope`——这是 dish 的**默认作用域**，由每个 dish 按自身实际
情况决定，不是独立类型；换源后用户看到的仍是上面三者之一。

每个 dish 通过 `scope_caps[3]` 声明各作用域的支持度：`Unknown`（未了解）、
`Unable`（不支持）、`Able_But_Not_Implemented`（支持但 chsrc 未实现）、
`Able_And_Implemented`（支持且已实现）。

### 安全规则

**1. 写操作前一律 `-dry`。** 这不是可选项。`chsrc set` 会改配置文件，`SystemScope`
在 Linux 写 `/etc`、在 Windows 由各 dish 委托给厂商命令行（如 `winget source add`）——
后者会留下难以逐条追溯的系统级改动。临时 `HOME` 只在 Linux/macOS 上有效；chsrc 的家目录在
Windows 上取 `USERPROFILE`（`lib/xy.h:1276-1277`），在 Linux/macOS 上取 `HOME`，
两者都不受 `HOME` 覆盖影响。

```bash
chsrc set -dry -scope=user pip      # 先看会发生什么
chsrc set -scope=user pip            # 再执行
chsrc get pip                        # 最后复查
```

`-dry` 覆盖绝大多数路径，但存在**已知缺口**：apt 系在旧格式源场景下会用裸 `fopen` 写
`/etc/apt/sources.list`，绕开 dry 守卫。涉及 debian/ubuntu 且系统没有 DEB822 源文件时，
`-dry` 仍可能落盘——此时需要额外的隔离手段。判定哪些场景会触发，见
`references/scope-policy.md` §七。

**2. 默认用 `-scope=user`。** 用户说「帮我换个源」时，多半指用户级全局，不是系统级。
升到 `SystemScope` 前先确认用户确实要全系统改，并说明影响面。

**3. 不要用容器隔离来保护宿主配置。** Windows 上改用 `-dry`；Linux/macOS 上可以叠加临时
`HOME`。

细节见 `references/scope-policy.md`。

---

## 六、换源后必须复查

```bash
chsrc set pip        # 换
chsrc get pip        # 查——这一步不能省
```

`get` 是只读的，随时可跑。换源后不跑 `get` 等于没验证——可能测速选了另一个源、
写入被权限拒绝、或配置被别的工具覆盖了。

出错时见 `references/recovery.md`。

---

## 七、Bootstrap（无编译器时的死锁场景）

如果 chsrc 没装、编译器没装、预编译二进制拿不到——**给编译器换源本身就需要一个换源工具**，
这就是死锁。上游为此提供 bootstrapper：先临时改用可直连的源把编译器装上，
**装完必须交接回 chsrc**。

当前上游仓库里只有 `Termux.bash` 一个 bootstrapper，其余平台尚未实现。

**关于「死锁」的一个重要限定：** 死锁成立的前提是「给编译器换源需要换源工具，而那个工具也没有」。
在网络通畅的环境里（如 GitHub 的常规 runner），`apt update` 直连官方源即可成功，编译器本来
就能直接装上——这个前提不成立。讨论或验证这个场景时先确认环境是否真的存在该前提，无法复现
就如实说明，不要把「bootstrapper 脚本能跑通」当成死锁被验证。详见
`references/bootstrap.md` 的「已知局限」。

```text
bootstrapper → 装上编译器 → make/just 构建 chsrc
             → chsrc --version 验证      ← 这一步不能省
             → 交接给 chsrc 做源治理      ← 这一步不能省
```

bootstrapper 只切**临时**源。用完不交接，等于把用户留在一个临时源上。
详见 `references/bootstrap.md`；要写自己的 bootstrapper 看 `references/bootstrap-authoring.md`。

---

## 参考文件

按需读取，不要一次全读：

| 文件 | 什么时候读 |
| :-- | :-- |
| `references/platform-detection.md` | 判断平台、架构、是否有 `just`/`make`/编译器 |
| `references/binary-resolution.md` | 装 chsrc 本身，或处理无预编译产物的平台 |
| `references/ecosystem-detection.md` | 用户说的东西对应哪个 dish |
| `references/source-governance.md` | `get`/`list`/`measure`/`set`/`reset` 的完整语义与退出码 |
| `references/scope-policy.md` | 作用域选择、权限、`-dry` 的确切行为 |
| `references/bootstrap.md` | 遇到无 chsrc 无编译器的死锁 |
| `references/bootstrap-authoring.md` | 需要写一个 bootstrapper |
| `references/recovery.md` | 换源失败、要回滚、配置被覆盖 |
| `references/collaboration-workflow.md` | 需要与 chsrc 上游协作、提 issue |
| `references/ci-validation.md` | 需要验证多平台行为 |

## 退出码

`0` OK / `1` Fatal / `2` Unknown（未知菜品或参数）/ `3` Unsupported /
`4` UserCause（用户侧问题，如没装某程序）/ `5` MaintainerCause / `6` ExternalError

退出码 `4` 通常意味着缺依赖——按 `references/recovery.md` 处理。