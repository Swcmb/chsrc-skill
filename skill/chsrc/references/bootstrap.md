# Bootstrap 死锁场景

## 进入条件

三个条件同时成立才进入 bootstrap 路线：

1. `chsrc` 不可用（`chsrc --version` 失败）
2. 目标平台拿不到预编译产物
3. 目标机器上没有可用的 C 编译器

上游为此单列了一节「预编译产生的死锁问题」（`bootstrap/README.md:27`），并指明目前只有
`Windows`、`Linux`、`macOS` 三个平台有预编译产物（`bootstrap/README.md:29`）。

## 死锁的本质

自行编译 chsrc 需要提前具备三项前提（`bootstrap/README.md:35-37`）：clone 或解压源码、
C 语言编译器、`GNU make` 或 `just`（上游标注非强制，有更好）。三项全部依赖包管理器，
而包管理器走哪个源本该由 chsrc 决定。

上游原文（`bootstrap/README.md:39`）：「**这是一把死锁，导致用户最终回到手动换源的原始农耕时代。**」
`README.md:293` 与 `README.md:332` 在 BSD 段和其他平台段各重复一次：「**如果还不存在这些依赖，
你将会被死锁住: 我还没有换源，我该如何安装这些依赖呢？**」

概括：**给编译器换源本身需要一个已经能换源的二进制工具，而那个工具要从源码编译出来。**

## bootstrapper 的职责边界

bootstrapper 是用**原生脚本语言**写成的脚本，每个称一个 `bootstrapper`
（`bootstrap/README.md:47`）。原生脚本语言指该平台开箱即有、无需额外安装。

它做两件事（`bootstrap/README.md:49-55`）：完成最基本换源让系统立刻能装别的软件；
再由用户自行决定是否继续装 chsrc——装则补最少依赖并编译安装，不装则流程结束。

关键约束：**最基本换源不测速**，由维护者暂时决定切到某一个具体源，先让系统跑起来
（`bootstrap/README.md:57`）。

因此 bootstrapper **只切临时源**。上游在 `bootstrap/README.md:85` 写明：「`chsrc-bootstrap`
只换到了某一个具体的源，只是临时性的，后续用户可能还想要换到其他源，此时 `chsrc` 就可以接管。」

上游还指出 bootstrapper 有副作用价值：它会让系统满足一些前置条件，`chsrc` 在这些前置条件
存在时接着换源，可大幅减轻 recipe 的实现难度（`bootstrap/README.md:87`）。

## 上游现状：只有一个 bootstrapper

`bootstrap/` 目录下实际只有两个文件：`bootstrap/README.md` 与 `bootstrap/Termux.bash`。
其余平台（Windows / Linux / macOS / BSD）**尚未实现**。上游全树 grep `bootstrap` 只在
`doc/10-如何编写recipe.md:30` 留了一条引用链接，13 个 CI workflow 里零命中。

上游对 BSD 的处境有专门说明（`bootstrap/README.md:95-104`）：FreeBSD 换源自项目一开始就是
目标之一，长期未实现的原因是换源步骤太多、面临特有的鸡蛋困境、BSD 工具与 GNU 工具存在不兼容、
C 语言中实现困难。`README.md:297` 补了更强的一条：`chsrc` 实现的 `FreeBSD recipe`
长期存在问题，因此一个新的 `bootstrapper` 是相当必要的。

## 标准流程

```text
bootstrapper  → 临时切到某个具体源（不测速）
             → 装上编译器 + make + git
             → make/just 构建 chsrc
             → chsrc --version 验证        ← 不能省
             → 交接给 chsrc 做源治理       ← 不能省
```

**`chsrc --version`** 在参数解析阶段被识别（`src/chsrc-main.c:228`），随后与 `version` /
`ver` / `-version` / `-v` 一同进入版本打印分支并返回 `Exit_OK`（`src/chsrc-main.c:265`）。
这一步验证三件事：二进制进了 `PATH`、编译产物可执行、版本宏正确。当前版本号
`Chsrc_Version` 为 `"0.2.7.2"`（`src/framework/version.h:11`）。

**交接**省掉等于把用户留在 bootstrapper 写死的那个临时源上，后续换源需求无处落地。
判断交接是否到位看 `chsrc get <dish>` 的输出，不看 bootstrapper 自己的成功提示。

## Termux.bash 逐段拆解

唯一现成 bootstrapper，用于 Termux 无预编译架构时引导。官方调用方式（`README.md:313-316`）：

```bash
curl -L https://gitee.com/RubyMetric/chsrc/raw/main/bootstrap/Termux.bash | bash
```

**① 优先走内置命令**（`bootstrap/Termux.bash:24-25`）：`termux-change-repo` 存在则直接调用，
交由 Termux 自带的交互式换源界面处理。

**② 降级路径：直接改文件**（`bootstrap/Termux.bash:32-39`）：内置命令不存在时，用 `sed` 就地改
三个 sources 文件，原有 `deb ... stable main` 行被注释掉并在其后追加镜像站行：

| 文件 | 目标 repo |
| :-- | :-- |
| `$PREFIX/etc/apt/sources.list` | `mirrors.cernet.edu.cn/termux/apt/termux-main` |
| `$PREFIX/etc/apt/sources.list.d/x11.list` | `mirrors.cernet.edu.cn/termux/apt/termux-x11` |
| `$PREFIX/etc/apt/sources.list.d/root.list` | `mirrors.cernet.edu.cn/termux/apt/termux-root` |

改完立即 `apt-get update`（`bootstrap/Termux.bash:33`）。三处硬编码同一站点 `mirrors.cernet.edu.cn`
——这正是「只切临时源」的体现。

**③ 更新并升级**（`bootstrap/Termux.bash:43`）：`apt-get update && apt-get upgrade`

**④ 交互询问**（`bootstrap/Termux.bash:47`）：`read -p "是否需要安装 chsrc ? (y/n): " ...`

**⑤ 装依赖 + 拉源码 + 构建**（`bootstrap/Termux.bash:51-58`）：

```bash
apt-get install -y gcc make git
git clone https://gitee.com/RubyMetric/chsrc.git --depth 1
cd chsrc
make build-in-release-mode
make install
```

`make build-in-release-mode` 是真实目标（`Makefile:200`），产物名由
`ReleaseMode-Target-Name = chsrc-release` 决定（`Makefile:85`）。`make install`（`Makefile:279`）
把二进制装到 `$(DESTDIR)/usr/bin/chsrc`，同时装 man page 与 bash 补全（`Makefile:280-282`）。

**上游脚本的一个缺口：** Termux.bash 走完 `make install` 就结束，缺 `chsrc --version` 验证，
也缺交回 chsrc 做源治理。照抄会漏掉标准流程的最后两步。

## 交接给 chsrc 的具体形态

Termux 是唯一有对应 bootstrapper 的 dish——`src/recipe/os/APT/Termux.c:23` 挂着一条全树唯一的
note「该 recipe 存在对应的 bootstrapper」。因此交接可直接落到命令：

```bash
chsrc set termux        # 换到 chsrc 治理的源
chsrc get termux        # 复查
```

依据：

- dish 名为 `termux`（`src/recipe/os/APT/Termux.c:5`），已在菜单登记（`src/recipe/menu.c:227`）
- `os_termux_setsrc()` 用与 bootstrapper 同款的 `sed` 手法改 sources.list，随后
  `apt-get update` + `apt-get upgrade`（`src/recipe/os/APT/Termux.c:62-68`）
- 候选源共 14 个：Upstream + USTC / SJTUG / TUNA / BFSU / PKU / NYIST / NJU / SUSTech /
  ISCAS / ZJU / SDU / CQUPT（`src/recipe/os/APT/Termux.c:26-38`）
- **cernet 不在候选源列表里**，它只出现在 `src/recipe/os/APT/Termux.c:51` 的一条注释链接中。
  所以 `chsrc set termux` 确实把系统从 bootstrapper 的临时源移走了

两个操作细节：该 recipe 用 `chef_set_os_scope`（`src/recipe/os/APT/Termux.c:18`），默认落
SystemScope；Termux 下 `chsrc_ensure_root()` 被显式注释掉（`src/recipe/os/APT/Termux.c:58`），
注释说明「Termux 下禁止使用 root」。写操作前先 `-dry`，选项必须写在 dish 名之前。

## 无现成 bootstrapper 时的处置

识别出「该平台无现成 bootstrapper」之后不要卡住，给 authoring 路径：

1. 用该平台**原生脚本语言**写 bootstrapper，无需额外安装
2. 只做两件事：最基本换源（不测速，切到某个具体源）+ 可选的装 chsrc
3. 保留交互询问，让用户自己决定要不要继续装 chsrc（`bootstrap/README.md:55`）
4. 上游有公开征集渠道，`README.md:295` 与 `README.md:334` 都指向 issue 230

写法规范见 `references/bootstrap-authoring.md`。原生脚本语言约束（`bootstrap/README.md:59-77`）：

| 平台 | 原生脚本语言 | 备选 |
| :-- | :-- | :-- |
| Windows | `PowerShell`, `CMD` | — |
| Linux | `Bash`, `sh` | `Perl` |
| macOS | `Zsh`, `Bash`, `sh` | `Ruby`, `Perl` |
| BSD | `sh` | 待议 |
| 其他平台 | 该平台支持的一切脚本语言 | 无需额外安装 |

备选项里的 `Perl` / `Ruby` 选之前先确认目标机装没装。

**写 bootstrapper 之前先确认其他路径真的走不通。** 上游对非 Termux 平台未提供 bootstrapper，
「有现成脚本可用」和「手写一个」是两件事。而上游给出的这两条路径都以「依赖已就位」为前提，
依赖缺失时它们就是死锁本身：

- BSD（`README.md:290`）：`clang -Iinclude -Ilib src/chsrc-main.c -o chsrc`
- 其他平台（`README.md:329`）：`git clone https://gitee.com/RubyMetric/chsrc.git; cd chsrc; make`

## 为什么天然属于隔离环境

判定标准就是「能否成功构建出 chsrc」。构建是测试目的本身。开发机通常已装编译器，
预存的 chsrc 也让场景前提失效。另外 bootstrapper 会改系统级配置文件
（`/etc/apt/sources.list`、Termux 的 `$PREFIX`），`make install` 写 `/usr/bin`，两条都有副作用；
构造「没有 chsrc」的干净前提也需要一次性容器或 runner。

因此：死锁复现放隔离环境；bootstrap 入口判定（识别平台、识别有无现成 bootstrapper、
给出 authoring 路径）与 handoff 判定属纯文本与静态事实，可以在本机做。

## 已知局限（务必诚实写出）

**GitHub 美欧 runner 上死锁前提不成立。** 该载体网络可直连官方源，`apt` 与 `curl` 齐备，
`apt update` 直连官方源即可成功。「要装编译器就得先换源」这个前提在该载体下不存在——
编译器本来就能直接装上。

因此测试断言目标被显式降维为：

> **「bootstrapper 能在无编译器环境下引导装上编译器并完成构建」**

这个断言测的是 bootstrapper 脚本的执行正确性与构建链路可用性，测不到死锁本身。

复现死锁需要构造的额外前提：

- 平台属于「无预编译产物」类。上游 `README.md` 的 BSD 段只给源码构建路径，没有任何二进制
  下载条目（`README.md:284-297`）；Linux 段列出 x64 / aarch64 / riscv64 / armv7 四个架构
  （`README.md:214-224`），这些架构是否在 release 上真正发布未核实
- 该平台的包管理器源处于不可直连或极慢状态
- 机器上确实没有编译器、没有 make / just

三条同时满足才能称为死锁复现。

**报告纪律：复现不出死锁就说复现不出。** 在 GitHub runner 上跑通 Termux bootstrapper 是一条
happy path。把它写成「成功破解了死锁」属于把顺利路径误报成目标达成。正确表述是
「bootstrapper 在该载体上执行成功，该载体不具备死锁前提，死锁场景未经复现」。

## 自查清单

- [ ] 记录了平台与架构，判定该平台是否有预编译产物
- [ ] `chsrc --version` 实际执行过并输出版本号
- [ ] 交接动作实际执行过（`chsrc set <dish>`），而非只给出建议
- [ ] `chsrc get <dish>` 复查过，当前源已脱离 bootstrapper 写死的那个源
- [ ] 无现成 bootstrapper 的平台给出了 authoring 路径而非终止
- [ ] 报告中区分了「bootstrapper 执行成功」与「死锁被复现」