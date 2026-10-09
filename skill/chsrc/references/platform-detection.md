# 平台与工具链检测

动手前先确定两件事：**运行 chsrc 的平台**，以及**有没有可用的构建工具链**。对应执行
流程的第 [2] 步。

铁律：**探测构建工具只查存在性，绝不调用。** 换源本身不做编译。存在即记录，能否构建
交给用户决定。

---

## 一、平台判定

### chsrc 自己怎么判

`_xy_detect_os()` 按**固定顺序**判定，首个命中即返回（`lib/xy.h:1550-1647`）：

| 顺序 | 依据 | 结果 | 出处 |
| :-- | :-- | :-- | :-- |
| 1 | 环境变量 `SystemDrive` + 该盘 `\Windows` 目录可打开 | `on_windows` | `lib/xy.h:1554-1571` |
| 2 | `/proc/version` 含 `Linux` | `on_linux` | `lib/xy.h:1574-1586` |
| 3 | 环境变量 `ANDROID_ROOT` 含 `/system` | `on_android` | `lib/xy.h:1593-1602` |
| 4 | `/System/Applications` 与 `/Library/Apple` 均可打开 | `on_macos` | `lib/xy.h:1605-1617` |
| 5 | `uname -s` 输出含 `BSD` | `on_bsd` | `lib/xy.h:1620-1632` |
| 6 | 第 5 步 `popen` 失败时，`/etc/rc.d` 可打开 | `on_bsd` | `lib/xy.h:1635-1643` |

五个标记位与 `os_family` 定义在 `lib/xy.h:81-106`。顺序有实际影响：Termux 在
`/proc/version` 上也含 `Linux`，会先被第 2 步截住（详见第四节）。

编译期另有一组宏（`lib/xy.h:111-128`），决定编译期分支（哪些 API 可用）：
`_WIN32` → `XY_Build_On_Windows`、`__linux__`/`__linux` → `XY_Build_On_Linux`、
`__APPLE__` → `XY_Build_On_macOS`、`__OpenBSD__`/`__NetBSD__`/`__FreeBSD__` →
`XY_Build_On_BSD`。运行期判定与这组宏互相独立。

### 手工探测命令

```bash
uname -s          # Linux / Darwin / FreeBSD / NetBSD / OpenBSD / MINGW64_NT-…
uname -m          # 架构
uname -o          # GNU/Linux vs Android；installer.sh 用它分 Termux
cat /proc/version # Linux 依据
echo "$ANDROID_ROOT"   # Termux/Android 依据
echo "$SystemDrive"    # Windows 依据（值为 C:）
echo "$OSTYPE"    # zsh/bash 变量，不在 chsrc 源码里，仅作交叉参考
```

### WSL

chsrc 源码没有任何 WSL 分支（无 `__wsl__`/`WSL_DISTRO_NAME`/`microsoft` 判断）。
WSL 里 `/proc/version` 含 `Linux`，被判成 `on_linux` 后按 Linux dish 处理，该路径
未在上游专门测试。按 `uname -r` 是否含 `microsoft` 自行识别 WSL1/WSL2，并向用户说明。

---

## 二、架构判定

### chsrc 运行时怎么判

`chsrc_get_cpuarch()`（`src/framework/OS.c:951-1004`）。

**Windows**（`src/framework/OS.c:957-979`）走 `GetNativeSystemInfo()` 读
`wProcessorArchitecture`：`AMD64 → x86_64`、`ARM64 → aarch64`、`ARM → arm`、
`INTEL → x86`、`IA64 → IA-64`；其它 / `UNKNOWN` 报「无法检测到CPU类型」，退出码 `4`。

**非 Windows**（`src/framework/OS.c:980-1002`）按序：`arch` 存在则执行 `arch`
（`src/framework/OS.c:984-989`），否则 `uname` 存在则执行 `uname -m`
（`src/framework/OS.c:991-996`），都没有则报错退出码 `4`。`arch` 优先于 `uname -m`。

### 架构到产物名的映射

`tool/installer.sh:164-184` 的 `set_arch()`：`x86_64 → x64`、`aarch64|arm64 → aarch64`、
`riscv64 → riscv64`、`armv7* → armv7`；其它报错转 bootstrapper 或自行编译。
Windows 侧由 `tool/installer.ps1:145-164` 用 `Get-WmiObject Win32_Processor` 的
`Architecture` 映射：`0 → x86`、`9 → x64`、`12 → arm64`，其它报「Unsupported
architecture」；URL 形如 `chsrc-{arch}-windows.exe`（`tool/installer.ps1:169`）。

**上游预编译产物的覆盖范围**（`README.md` §安装 逐段核实）：

| 平台 | 有预编译产物的架构 |
| :-- | :-- |
| Windows | x86 / x64 / arm64 |
| Linux | x64 / aarch64 / riscv64 / armv7 |
| macOS | x64 / arm64 |
| BSD | **无** |

只有 BSD 与 Android/Termux 拿不到可用的官方预编译包，走源码构建或 bootstrapper。
产物名与 URL 见 `references/binary-resolution.md`。

---

## 三、构建工具探测

### 只查存在性

chsrc 自带的探测是 `chsrc_check_program_quietly()`（`src/framework/OS.c:216-220`），
底层 `cmd_to_check_program()`（`src/framework/OS.c:166-174`）在 Windows 用 `where`、
其余用 `command -v`。选 `command -v` 而非 `which`/`whereis` 的原因写在
`src/framework/OS.c:162-164`：部分 Linux 发行版不自带那两个命令。

外部手工探测只看退出码，**不要执行这些工具**：

```bash
for t in just make cc gcc clang curl perl; do
  printf '%-6s ' "$t"
  command -v "$t" >/dev/null 2>&1 && echo yes || echo no
done
```

`curl` 是测速的硬依赖：`dish_select_fastest_source()` 先探测它，缺失即报错退出
（`src/framework/source.c:380-386`），另有针对 Cygwin 版 curl 已知 bug 的专门检查
（`src/framework/source.c:391-403`）。少了 curl，`measure` 与不带源名的 `set` 都无法工作。

### justfile 与 Makefile 的分工

`justfile` 头注释写明它服务于**原生 Windows**，不依赖 GNU make 及 MSYS2/Cygwin
环境（`justfile:11-14`），并显式设置 Windows shell（`justfile:32-33`）：

```
[windows]
set shell := ['cmd', '/c']
```

这个 `[windows]` 块同时是「本机在 Windows 上」的判据。编译器按平台自动选
（`justfile:35-41`）：Windows → `gcc`，macOS → `clang`，其余 → `cc`；DEBUGGER 分
Windows/macOS → `gdb`/`lldb`（`justfile:43-49`）。clang 在 Windows 上追加
`-target <arch()>-pc-windows-gnu` 以适配原生 WoA 环境（`justfile:52-55`）。

`Makefile` 走 `uname` 与 `$(OS)` 判断平台（`Makefile:18-33`）：`uname` 为 `Linux`、
`Darwin` 分别置 `On-Linux`、`On-macOS`，`$(OS)` 等于 `Windows_NT` 置 `On-Windows`。
注释说明只有 MSYS2 定义 `$(OS)` 而原生 Windows 不定义，反之原生 Windows 定义
`$(ComSpec)`（`Makefile:29-30`）。默认目标在 `Makefile:145-149`
（`b`/`bd`/`br`/`bcir`）。**justfile 里没有 CI 目标**，GitHub Actions 走 make
（`justfile:27-29`）；`bcir` 只在 Makefile 中存在。

### 本机工具链现状

在本仓库工作目录实测（2026-10）：

| 工具 | 状态 |
| :-- | :-- |
| `just` | 缺失 |
| `make` | 缺失 |
| `clang` | 缺失 |
| `cc` | 存在，`/d/ProgramFiles/msys64/ucrt64/bin/cc` |
| `gcc` | 存在，`/d/ProgramFiles/msys64/ucrt64/bin/gcc` |
| `perl` | 存在，`/usr/bin/perl` |
| `curl` | 存在，`/mingw64/bin/curl` |

本机既无 `just` 也无 `make`，源码构建路径走不通。换源改用**已安装的 chsrc 二进制**
或官方预编译产物。即便 `gcc` 在 PATH 中，本篇探测也只确认存在性，不触发编译。
（`gcc` 存在与否决定 `just b` 能否跑，前提是先有 `just`。）

---

## 四、Termux / Android

### 判定的坑

`/proc/version` 在 Termux 上含 `Linux`，而 Linux 判定（`lib/xy.h:1574-1586`）排在
Android 判定（`lib/xy.h:1593-1602`）之前，命中即 `return`。所以 Termux 被标成
`on_linux`，`on_android` 保持 false。Android 判定读 `ANDROID_ROOT` 并要求含
`/system`，依据注释指向 AOSP 的 `init.environ.rc.in`（`lib/xy.h:1591`）。

`tool/installer.sh:186-220` 的 `set_platform()` 走另一条路：`uname -s` 得 `linux` 时
再用 `uname -o` 判定，若为 `android` 则**不下载预编译产物**，改为提示并直接跑
bootstrapper：`wget -O - .../bootstrap/Termux.bash | bash`
（`tool/installer.sh:193-200`）。

### bootstrapper

`bootstrap/Termux.bash` 是仓库里唯一的 bootstrapper：`termux-change-repo` 存在则直接
调用（`bootstrap/Termux.bash:24-25`）；否则用 `sed` 改 `$PREFIX/etc/apt/sources.list`
加 `mirrors.cernet.edu.cn/termux/apt/termux-main`，并处理 x11 / root 两个附加源
（`bootstrap/Termux.bash:32-39`）；接着 `apt-get update && apt-get upgrade`；最后询问
是否装 chsrc，是则 `apt-get install -y gcc make git`、clone、
`make build-in-release-mode`、`make install`（`bootstrap/Termux.bash:49-59`）。

Termux 前缀常量：`#define OS_Termux_SourceList "/data/data/com.termux/files/usr"` 接
apt 的 sources.list 路径（`src/recipe/os/APT/common.h:40`）。Termux dish 的
`chsrc_ensure_root()` 被注释掉并注明「Termux下禁止使用root」
（`src/recipe/os/APT/Termux.c:58`）。

---

## 五、BSD 三系差异

三系各有独立 dish，共享 `chef_set_os_scope()` 与 SystemScope 定位，但换源目标文件、
镜像布局、可用动作都不同。

| | FreeBSD | NetBSD | OpenBSD |
| :-- | :-- | :-- | :-- |
| dish | `freebsd` | `netbsd` | `openbsd` |
| 配置文件 | `/usr/local/etc/pkg/repos/*.conf` | `/usr/pkg/etc/pkgin/repositories.conf` | `/etc/installurl` |
| 上游 | `pkg.freebsd.org` | `ftp.netbsd.org/pub/pkgsrc/packages/NetBSD/` | `cdn.openbsd.org/pub/OpenBSD/` |
| 英文输出 | 拒绝 | 拒绝 | 拒绝 |
| 用户自定义源 | 拒绝 | 默认允许 | 允许 |
| 声明支持的操作 | get/set | get/set | get/set |

出处：FreeBSD `src/recipe/os/BSD/FreeBSD.c:5`、`:18-21`；NetBSD
`src/recipe/os/BSD/NetBSD.c:5`、`:18-20`；OpenBSD `src/recipe/os/BSD/OpenBSD.c:6`、
`:18-21`。

三系的 `setsrc` 都先 `chsrc_ensure_root()`（FreeBSD `src/recipe/os/BSD/FreeBSD.c:46`、
NetBSD `src/recipe/os/BSD/NetBSD.c:57`、OpenBSD `src/recipe/os/BSD/OpenBSD.c:56`）。
FreeBSD 注释特别说明它不自带 sudo 仍要求 root（`src/recipe/os/BSD/FreeBSD.c:45`）。

FreeBSD 有五类源：pkg、ports、port、portsnap、update，其中 portsnap 在 FreeBSD 14
已被移除（`src/recipe/os/BSD/FreeBSD.c:40`）；源列表要求 Ustc 与 Nju 镜像至少保留一个、
Nju 前有至少一个镜像（`src/recipe/os/BSD/FreeBSD.c:25` 注释）。NetBSD 默认没装 pkgsrc
（`src/recipe/os/BSD/NetBSD.c:52` 注释）。

`cp --backup` 在 BSD 与 macOS 上不支持，备份逻辑已按平台分支处理
（`src/framework/OS.c:906`）。installer.sh 把 `bsd` 与 `dragonfly` 归到同一
`platform=bsd` 并告知无预编译产物（`tool/installer.sh:204-212`）；Dragonfly 没有独立
dish。

---

## 六、未知平台的兜底

全部标记位都为 false 时，chsrc 直接 `xy_throw ("Unknown operating system")`
（`lib/xy.h:1645-1646`）。`xy_throw` 实现是 `assert(!reason)`（`lib/xy.h:140`）——
在 `NDEBUG` 生效的构建里 assert 被关闭，该路径静默通过。**别把这条路径的静默当作
「已识别」。**

判定失败时的处理顺序：

1. 逐项手工复核第一节的探测命令，列出实际输出。
2. 检查是否 WSL——WSL 走 Linux 分支，属已知未专门测试的路径。
3. 都不匹配则报告「平台未知」，说明已排除 Windows / Linux / Android / macOS /
   BSD。不要猜一个平台继续。
4. 已知平台的预编译产物情况见 `references/binary-resolution.md`。

退出码 `4`（`Exit_UserCause`）表示用户侧缺依赖，常见于缺 `curl` 或架构检测失败
（`src/framework/OS.c:977`、`src/framework/OS.c:1001`）。处理见
`references/recovery.md`。

---

## 七、探测结果怎么用

| 探测结果 | 换源影响 |
| :-- | :-- |
| 有 chsrc 二进制 | 直接用，与平台无关。**默认路径** |
| 无 chsrc，有预编译产物 | 装二进制，见 SKILL.md 第三节 |
| 无 chsrc，无产物（BSD / arm64 / riscv64 Linux） | 源码构建或 bootstrapper；需要构建工具链 |
| 缺 `curl` | 测速不可用。`set` 必须显式给镜像站 code 或 URL |
| 平台未知 | 停止并报告，见第六节 |

平台与架构只影响**装 chsrc 本身**这一步。已装好的 chsrc 在任何平台上换源的操作完全
一致——选项前置（`chsrc get -no-color pip`）、`-dry` 预览、`-scope` 选择、`set` 第二
参数指定镜像站。