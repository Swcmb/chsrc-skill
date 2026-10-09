# 跨平台验证方法（记录，不执行）

**本文件是方法记录。** 记录上游 chsrc 用什么方式在哪些平台上验证自己的构建产物，
供需要判断「某个平台的行为有没有被验证过」时查阅。

**本 Skill 在任何环境都不执行构建。** 本项目不改上游源码、不跑 `make` / `just`、
不编译二进制。所有验证命令都只作为「交给用户在自己机器上跑」的标准清单给出。
判断某个平台是否可信时，看下面的矩阵；判断某个命令是否存在时，看 `references/platform-detection.md`。

---

## 一、上游 CI 矩阵

上游 `.github/workflows/` 下共 **13 个 workflow 文件**，其中 **7 个是构建矩阵**，
产物统一上传到 GitHub 的 `pre` release：

| workflow 文件 | 目标平台 | 产物名 |
| :-- | :-- | :-- |
| `build-on-Linux-x64.yml` | Linux x64 | `chsrc-x64-linux` |
| `build-on-Linux-AArch64.yml` | Linux AArch64 | `chsrc-aarch64-linux` |
| `build-on-Linux-ARMv7.yml` | Linux ARMv7 | `chsrc-armv7-linux` |
| `build-on-Linux-riscv64.yml` | Linux riscv64 | `chsrc-riscv64-linux` |
| `build-on-macOS.yml` | macOS arm64 + macOS x64 | `chsrc-aarch64-macos` / `chsrc-x64-macos` |
| `build-on-Windows.yml` | Windows x64 / x86 / Android arm64 | `chsrc-x64-windows.exe` / `chsrc-x86-windows.exe` / `chsrc-arm64-android` |
| `build-on-Windows-Arm64.yml` | Windows on Arm64 | `chsrc-arm64-windows.exe` |

其余 6 个 workflow 与构建矩阵无关：`pkg-deb.yml`（打 deb 包）、
`PR-test.yml`、`PR-notify.yml`、`pub-AUR-chsrc-and-chsrc-bin.yml`、
`pub-AUR-chsrc-git.yml`、`pub-WinGet.yml`（发布到 AUR / WinGet）。

7 个构建 workflow 的触发条件一致：**push 到 `gh-build` 分支**
（如 `.github/workflows/build-on-Linux-x64.yml:14-15`）。`gh-build` 由维护者在发版时推送，
普通开发分支的 push 不触发产物构建。

### 实际使用的 runner

```text
ubuntu-latest        x64 / AArch64 / ARMv7 / riscv64 / deb
macos-latest         macOS arm64
macos-15-intel       macOS x64
windows-latest       Windows x64 / x86 / Android arm64
windows-11-arm       Windows Arm64（原生 WoA）
```

非 x86 架构的 Linux 产物在 `uraimo/run-on-arch-action@v3` 的容器里交叉/模拟构建，
`arch` 分别取 `aarch64` / `armv7` / `riscv64`，`distro: ubuntu_latest`
（`.github/workflows/build-on-Linux-AArch64.yml:22-25`）。

Windows Arm64 走 MSYS2 的 `CLANGARM64` 环境装原生 aarch64 clang 工具链
（`.github/workflows/build-on-Windows-Arm64.yml:31-39`）。

### BSD：PLATFORM_UNAVAILABLE

GitHub 托管 runner 的镜像只有 ubuntu / macos / windows 三系，**没有任何 BSD runner**。
仓库内 13 个 workflow 里也没有一处 BSD 目标——BSD 相关字样只出现在
`.github/ISSUE_TEMPLATE/01-Report_Bug.yml:61-63`（FreeBSD / NetBSD / OpenBSD 的 bug 模板选项）。

因此 **FreeBSD / NetBSD / OpenBSD 上的 chsrc 行为未经 CI 验证**。在 BSD 上回答任何
「这个版本支持得怎么样」的问题时，标记为 `PLATFORM_UNAVAILABLE`：
说明上游 CI 覆盖不到该平台，结论只能来自源码阅读或用户在 BSD 上的实测。

同理，Termux / Android 的验证只覆盖到 `chsrc-arm64-android` 这一个产物
（`.github/workflows/build-on-Windows.yml:55-70`，在 Windows runner 上用 NDK 的
`aarch64-linux-android21-clang` 交叉编译，带 `CROSS_BUILD_WINDOWS_FOR_ANDROID=1`）。

---

## 二、CI 只走 make

**7 个构建 workflow 全部调用 `make`，没有一个调用 `just`。** 命令一律是：

```bash
make build-in-ci-release-mode
```

Windows 两个 workflow 调的是 MSYS2 里的 `mingw32-make.exe`（`.github/workflows/build-on-Windows.yml:42`、
`.github/workflows/build-on-Windows-Arm64.yml:43`）。

`justfile` 刻意不实现 CI 目标，文件头的注释写明原因（`justfile:28-29`）：

> 注意，由于我们要在 GitHub Actions 上编译 x32 版的 chsrc，所以需要使用 make，
> 而不清楚 just 在 MINGW32 中的情况，所以我们在此 justfile 中并不实现关于 CI 的功能

对应地，`justfile` 里只有 `build-in-dev-mode` / `build-in-debug-mode` /
`build-in-release-mode` 三个产物名（`justfile:69-71`），没有 `chsrc-ci-release`，
没有 `bcir`，没有任何 CI 专用 recipe。

**唯一的例外是 `PR-test.yml`**：它在 Windows 上用 `choco install just` 装 `just`，
然后跑 `just` 和 `just test`（`.github/workflows/PR-test.yml:62-72`）；
Ubuntu 上则跑 `make` 和 `make test`（`.github/workflows/PR-test.yml:39-45`）。
这属于 PR 冒烟测试，不是产物构建，仍然没有触及 justfile 缺失的 CI 目标。

---

## 三、Makefile 的 CI-RELEASE 模式

`build-in-ci-release-mode` 只在 `Makefile` 里存在，别名 `bcir`（`Makefile:149`）。
它是 CI-RELEASE 模式，产物名 `chsrc-ci-release`（`Makefile:86`）。

与普通 `br`（`build-in-release-mode`）的差异有两处：

| 差异 | 位置 | 说明 |
| :-- | :-- | :-- |
| Linux 静态链接 | `Makefile:126-128` | `ifeq ($(On-Linux), 1)` 时追加 `-static` |
| macOS 固定 gcc-14 | `Makefile:133-135` | `CC = gcc-14`，注释说明 GitHub Actions 上的 LLVM 太老、gcc 被改名成 clang |

macOS x64 固定用 `macos-15-intel`，注释记录了原因
（`.github/workflows/build-on-macOS.yml:51-52`）：`macos-13` 已于 2025-12-04 下线。

Linux 静态链接之外的平台保持动态链接，所以 CI-RELEASE 产物在各平台上链接行为不同，
比对文件大小时要用这一点解释差异。

### Windows 交叉构建的两个坑

**x32 需要先删 `chsrc.res`。** workflow 在切到 `MINGW32` 前显式 `rm -f chsrc.res`
（`.github/workflows/build-on-Windows.yml:45-53`），注释说明了原因：残留的 x64 资源文件
会让 make 误判它已是最新而不重新生成，最终报 `file format not recognized`。

**Android 构建必须关掉 Windows 资源。** `Makefile:110-115` 用
`CROSS_BUILD_WINDOWS_FOR_ANDROID` 变量排除 `Use-Binary-Windows-Resource`，
Android 交叉编译时该变量置 1。

手工在 Windows 上复现 x32 构建时，这两个变量和那一步 `rm` 都要照抄。

---

## 四、已知局限：CI 载体上的死锁前提不成立

上游 `references/bootstrap.md` 描述的死锁场景要求「装编译器的源本身连不上」。
**GitHub 美欧 runner 上这个前提不可复现**：runner 网络畅通，
`apt-get update -qq` 直连官方源即成功（`.github/workflows/build-on-Linux-AArch64.yml:30-32`
等三个非 x86 workflow 的第一步都是裸 `apt-get`）。

结论：**CI 里跑出来的是 bootstrapper 的 happy path**，
即源通畅时的正常路径。源不通时的行为（`apt` 超时、回落、bootstrapper 中途失败、
交接回 chsrc 的时机）都没有被上游 CI 覆盖过。

判断 bootstrapper 是否可靠时，不要引用 CI 绿灯作为证据。要评估失败路径，
只能读 `references/bootstrap-authoring.md` 的流程约定，或让用户在源确实不通的环境里实测。

同理，PR-test.yml 的 `make test` / `just test` 只跑 `test-xy` 和 `test-fw`
两个单元级程序（`Makefile:218`、`justfile:136`）。CLI 测试 `test/cli.pl`
只在 lefthook 的 `pre-push` 钩子里跑（`lefthook.yml:45-53`，仅 `gh-build` 分支），
不进 CI workflow。

---

## 五、交给用户自己验证的标准命令

用户在自己的机器上验证时，给下面的命令，保持走构建系统。
**不要给绕过 `make` / `just` 直接敲编译器的野路子**——那会跳过 `.res` 资源生成、
`-DXY_DEBUG` 之类的条件编译，以及 Windows/macOS 的编译器选择逻辑，
测出来的结果不能代表上游行为。

四种构建模式（`doc/01-开发与构建.md:49-56`）：DEV / DEBUG / RELEASE / CI-RELEASE，
前三种日常用，第四种只在 GitHub Actions 用。

| 目的 | make | just |
| :-- | :-- | :-- |
| DEV → `chsrc` | `make b` | `just b` |
| DEBUG → `chsrc-debug` | `make bd` | `just bd` |
| RELEASE → `chsrc-release` | `make br` | `just br`（静态链接 `just STATIC=1 br`） |
| CI-RELEASE → `chsrc-ci-release` | `make bcir` | 无（justfile 不提供） |
| 覆盖编译器 | `make CC=gcc br` | `just CC=clang br` |
| 启动调试器 | `make debug` | `just debug` |
| 单元测试 | `make test` | `just test` |
| CLI 测试 | `make test-cli` | `just test-cli` |
| 快速 CLI 检查 | `make fastcheck` | `just fastcheck` |
| 清理 | `make clean` | `just clean` |

DEBUG 二进制是验证 recipe 的主力：`-DXY_DEBUG` 下执行任意 get/set/reset 会把
**所有** dish 的 `_prepare()` 跑一遍，prepare 有问题会立刻崩
（`doc/01-开发与构建.md:60`）。改完 `src/` 下的 `.c` / `.h` 后的标准动作是
`make bd && ./chsrc-debug get <受影响的 dish>`。

**Windows 原生**优先用 `just`——`justfile` 头部说明它就是为「原生 Windows 上不借助
GNU make 与 MSYS2 / Cygwin 环境」写的（`justfile:13-14`），且 `[windows]` 段把
shell 设为 `cmd /c`（`justfile:32-33`）。走 `make` 时注意 RELEASE 模式会依赖
`windres` 生成 `chsrc.res`（`Makefile:190-194`），`windres` 不在 PATH 上就会失败。

**macOS** 上 `make br` 默认 `CC = clang`、`DEBUGGER = lldb`（`Makefile:45-52`）；
DEBUGGER 可用 `make debug DEBUGGER=lldb` 覆盖。

测速依赖外部 `curl`。缺 curl 时 `measure` 相关操作会直接报错退出，
详见 `references/recovery.md`（退出码 `4`）。

---

## 六、本文件与上游的对应关系

改动上游构建配置后，上游这份记录就过时了。重新核对时按这个顺序 grep：

```text
.github/workflows/     13 个 workflow，7 个构建；runs-on 与触发分支
Makefile:106,123,149   bcir 与 build-in-ci-release-mode
Makefile:110-137       平台差异（静态链接、gcc-14、Android 交叉）
justfile:28-29         「不实现 CI 功能」的注释
lefthook.yml           pre-commit / post-merge / pre-push 各自的钩子与分支
```

上游的 `CLAUDE.md` 与 `doc/01-开发与构建.md` 是权威来源，本文件只做检索入口。
