# 编写 bootstrapper

当 chsrc 没有预编译产物、机器上也没有编译器时，bootstrapper 是唯一的破局路径。上游对
bootstrapper 的定义见 `bootstrap/README.md:47`，仓库 `bootstrap/` 目录目前只放了
`bootstrap/Termux.bash` 一个实现，其余平台待补（`bootstrap/README.md:104`）。

写新 bootstrapper 前先读 `references/bootstrap.md`——那篇讲怎么**用**已有脚本，本篇讲怎么**写**。

---

## 一、触发条件

同时满足三条才值得写：

1. 该平台/架构在上游无预编译产物（`tool/installer.sh:203` 对 `darwin` 之外的系统给出
   「无预编译产物」的分支，`bsd|dragonfly` 明确落到 bootstrap 提示）
2. 编译 chsrc 所需的 C 编译器缺失——这正是 bootstrap README 描述的死锁：
   装依赖需要包管理器，用包管理器需要先换源（`bootstrap/README.md:33-39`）
3. 该平台有原生脚本语言，用户不用额外装任何东西就能跑（`bootstrap/README.md:77`）

反过来说，用户已经有编译器时，直接给编译命令即可，别塞一个 bootstrapper
（`README.md:329`；BSD 手工编译命令见 `README.md:290`：`clang -Iinclude -Ilib src/chsrc-main.c -o chsrc`）：

```bash
git clone https://gitee.com/RubyMetric/chsrc.git; cd chsrc; make
```

---

## 二、语言约定

上游对每个平台指定了唯一允许的脚本语言集合（`bootstrap/README.md:61-77`）：

| 平台 | 允许 | 备选 |
| :-- | :-- | :-- |
| Windows | `PowerShell`、`CMD` | — |
| Linux | `Bash`、`sh` | `Perl` |
| macOS | `Zsh`、`Bash`、`sh` | `Ruby`、`Perl` |
| BSD | `sh` | 待议 |
| 其他平台 | 该平台自带的一切脚本语言 | 无额外要求 |

扩展名按语言取：`.bash` / `.sh` / `.ps1` / `.cmd`，放 `bootstrap/` 目录根部。
`Termux.bash` 的 shebang 是 `#!/usr/bin/env bash`（`bootstrap/Termux.bash:1`），
文件头带 SPDX 标识、`File Name`、`File Authors`、`Created On`、`Last Modified`——
新脚本照抄这个头部结构（`bootstrap/Termux.bash:2-18`）。

---

## 三、最小职责：三段 + 交棒

上游对 bootstrapper 的职责定义是**两件事**（`bootstrap/README.md:47-55`）：

1. 用一个确定可直连的镜像完成最基本的换源，让用户立刻能装别的软件
2. 按用户意愿装最少依赖并编译安装 chsrc；用户拒绝时流程直接结束

由此推出三段结构：

```text
探测缺什么  →  临时切到可直连的源  →  装上（编译器 / 构建工具 / 依赖）
           →  打印 chsrc 验证命令 + 后续换源步骤   ← 交棒，不能省
```

**临时源不测速。** 上游明确写「最基本的换源，即不测速，由维护者暂时决定先切换到某一个
具体的源」（`bootstrap/README.md:57`）。脚本里写死一个稳定镜像 URL，并在注释里注明依据
（`bootstrap/Termux.bash:15` 的 `@consult` 链接就是干这个的）。

---

## 四、参考实现拆解：`bootstrap/Termux.bash`

共 61 行，逐段对照：

| 段 | 行号 | 做了什么 |
| :-- | :-- | :-- |
| 前置守卫 | `:24` | `command -v termux-change-repo &>/dev/null`——官方交互工具在就直接用它，脚本自己不动文件。**这是幂等的第一道闸** |
| 改 termux-main 源 | `:32` | `sed -i 's@^\(deb.*stable main\)$@#\1\ndeb <镜像> apt/termux-main stable main@'` |
| 改 x11 / root 源 | `:36` `:39` | 同样的手法，作用于 `$PREFIX/etc/apt/sources.list.d/{x11,root}.list` |
| 刷新 | `:33` `:43` | `apt-get update`，末尾 `apt-get update && apt-get upgrade` |
| 确认 | `:47` | `read -p "是否需要安装 chsrc ? (y/n): "` |
| 装依赖 | `:51` | `apt-get install -y gcc make git` |
| 取源码 | `:52` | `git clone https://gitee.com/RubyMetric/chsrc.git --depth 1` |
| 编译安装 | `:57-58` | `make build-in-release-mode` + `make install` |
| 收尾 | `:59` | 只打印「chsrc 安装完成!」——**没有验证命令，没有交接指引** |

第 59 行是这份参考实现最该被改进的地方。新脚本在这里补上 `chsrc --version` 与换源提示。

### 值得抄的三处

1. **输出前缀**——`bs_echo()` 统一加 `[chsrc-bootstrap]`（`bootstrap/Termux.bash:20-22`），
   用户一眼分清哪几行来自 bootstrapper。
2. **sed 的幂等技巧**——`:32` 把命中的旧行替换成 `#<原行>` 加新行。正则锚定
   `^\(deb.*stable main\)$`，注释行以 `#` 开头，第二次运行匹配不上，重复执行不会插入第二条
   镜像行。**注释式替换是这类脚本唯一的幂等保证**，删掉原行就会破坏它。
3. **存在性守卫先行**——`:24` 先查官方工具再决定是否动手，比无条件 sed 稳。

---

## 五、骨架

Bash / sh / zsh 版本（Windows 用 `.ps1`，语义同构）：

```bash
#!/usr/bin/env bash
set -euo pipefail                      # 任一命令失败立即非零退出
# 头部：SPDX / File Name / Authors / Created On / Last Modified
# @consult 镜像选择依据的链接

SRC_LIST="/etc/apt/sources.list"        # 平台实际的源文件
BACKUP="${SRC_LIST}.bak"

bs_echo() { echo "[chsrc-bootstrap] $*"; }

# 1 探测：缺什么装什么
for prog in gcc make git; do
  command -v "$prog" >/dev/null 2>&1 || MISSING="$MISSING $prog"
done

# 2 备份 + 临时切源（注释式替换保持幂等）
[ -f "$BACKUP" ] || cp "$SRC_LIST" "$BACKUP"
sed -i 's@^\(deb .*\)$@#\1\ndeb <可直连镜像> stable main@' "$SRC_LIST"
apt-get update

# 3 装依赖 + 编译安装 chsrc
apt-get install -y gcc make git
git clone https://gitee.com/RubyMetric/chsrc.git --depth 1
cd chsrc
make build-in-release-mode              # Makefile:200
sudo make install                      # Makefile:279-282，装到 /usr/bin

# 4 交棒（必写）
bs_echo "chsrc 已安装。现在请验证并接管："
echo "  chsrc --version                 # 验证"
echo "  chsrc ls                        # 列出可用镜像站 code"
echo "  chsrc set -dry -scope=user pip  # 预览换源"
echo "  chsrc set -scope=user pip       # 执行"
bs_echo "bootstrapper 只切了临时源，长期源治理交给 chsrc。"
```

无 `make` 时的等价编译命令（`tool/installer.sh:105`）：

```bash
cc/gcc/clang -Iinclude -Ilib src/chsrc-main.c -o chsrc
```

`-Iinclude -Ilib` 是硬要求，Makefile 自身也是这么加的（`Makefile:61`）。

---

## 六、交棒：脚本的最后一屏

结束前必须打印三样东西，缺一样都算未完成：

1. **验证命令** `chsrc --version`——它映射到 `cli_print_version()`，成功返回 `Exit_OK`
   （`src/chsrc-main.c:262-271`，实现见 `src/framework/waiter.c:449-455`）
2. **后续换源入口** `chsrc ls`（查镜像站 code）+ `chsrc set -dry <dish>` 预览
3. **临时源声明**——上游设计里 bootstrapper 只负责让系统跑起来，长期源治理由 chsrc 接管
   （`bootstrap/README.md:85`）。这句话写进输出，用户才不会停在临时源上。

写完 chsrc 版本的 bootstrapper 后，同步在对应 recipe 上加
`chef_set_note(this, "该 recipe 存在对应的 bootstrapper", ...)`
（`src/framework/chef.c:438`，范例见 `src/recipe/os/APT/Termux.c:23`）。

---

## 七、语言无关的共性铁律

| 要求 | 落地方式 |
| :-- | :-- |
| **幂等** | 改前 `command -v` / `grep -q` 判重；sed 用注释式替换而非插入式替换；重复执行结果一致 |
| **失败非零退出** | `set -euo pipefail`（sh 用 `set -eu`）；每条外部命令检查返回码，参照 `tool/installer.sh:54` 的 `error_exit` |
| **不改用户的长期配置** | 换源前 `cp -f <file> <file>.bak`（chsrc 自己就这么做，`src/framework/OS.c:872-919`）。原内容以注释或备份保留，随时可还原 |
| **错误不静默** | 失败时把实际执行的命令、返回码、文件路径打到 stderr，再退出 |
| **零依赖** | 只用系统自带工具。下载工具在 curl / wget 之间二选一（`tool/installer.sh:322-339`） |
| **可 dry 预览** | chsrc 有 `-dry`（`src/chsrc-main.c:211-214`），脚本内部的写操作没有这层保护。给脚本加 `--dry` 分支，或至少在改配置前 echo 出将要执行的命令 |

---

## 八、常见坑

**非交互 shell 下的确认提示。** 脚本常被 `curl -L <url> | bash` 执行（`README.md:316`），
stdin 被脚本内容占满，任何 `read` / 交互提示要么读到 EOF 要么挂死。`bootstrap/Termux.bash:47`
的 `read -p` 正是这个形态：管道执行时变量取到空值，装 chsrc 的分支会被跳过。修法是检查
`read -r x` 的退出码，EOF 时走默认分支并提示改用 `bash <(curl -L <url>)`。同理
`apt-get upgrade`（`bootstrap/Termux.bash:43`）缺 `-y`，非交互下会中止。`apt` 需 `-y`，
`pacman` 需 `--noconfirm`，`dnf/yum` 需 `-y`，`zypper` 需 `--non-interactive`。

**root / sudo。** 提权集中在**单条**命令前缀，脚本自身不切身份——非交互环境里任何交互式
密码输入都会卡死。chsrc 侧的规则可作参照：`chsrc_ensure_root()` 检测失败即
`exit(Exit_UserCause)`，即退出码 `4`（`src/framework/OS.c:279-301`，`src/framework/base.c:162-167`）；
Termux recipe 明确禁用 root（`src/recipe/os/APT/Termux.c:58` 的注释「Termux下禁止使用root」）。

`make install` 目标是 `/usr/bin/chsrc` + man page + bash completion
（`Makefile:279-282`），必然需要 root。非 root 场景改用 `install -D -m 755 chsrc ~/.local/bin/chsrc`
（参照 `tool/installer.sh:26` 的 `default_install_dir_4nonroot`）。

**架构探测。** 别只认一种来源：POSIX 用 `uname -m`（`tool/installer.sh:147-149`）再映射成
chsrc 的命名（`x86_64→x64`、`aarch64|arm64→aarch64`、`riscv64`、`armv7*`，
`tool/installer.sh:164-183`）；Windows PowerShell 用 `Get-WmiObject Win32_Processor` 的
`Architecture`（`0→x86`、`9→x64`、`12→arm64`），`9` 还要用 `Is64BitOperatingSystem` 二次判定
（`tool/installer.ps1:143-163`）；chsrc 内部是 `arch` 优先、回退 `uname -m`、都没有则报错退出
（`src/framework/OS.c:951-1004`）。探测失败要显式报错并给出手工编译命令，别静默选猜测值——
`tool/installer.sh:172-181` 的 `set_arch` 默认分支就是这么做的。

**程序存在性检查的跨平台写法。** `which` / `whereis` 并非所有发行版都有，`where` 仅 Zsh 可用。
chsrc 统一用 Windows `where` + Unix `command -v`（`src/framework/OS.c:159-174`），照此写。

**git 浅克隆的版本丢失。** `--depth 1`（`bootstrap/Termux.bash:52`）只取默认分支，仓库里
没有 tag。需要钉住某个 release 时用 `git clone -b <tag> --depth 1`。

**测速依赖 curl。** chsrc 的 `measure` 依赖外部 `curl`，缺失即报错退出
（`src/framework/source.c:380`）。可能引导用户跑 `chsrc measure` 的平台，把 curl 一并装上。

---

## 九、自检清单

连续跑两次输出「已是最新的镜像配置」且不产生第二条镜像行；故意断网制造一次失败，确认退出码
非零、错误信息可见；原始配置文件有 `.bak` 可还原；末屏含 `chsrc --version` 与 `chsrc set -dry`
提示；声明临时源性质；对应 recipe 已加 `chef_set_note`；头部元信息完整且 `Last Modified` 已填。

---

## 十、提交上游

上游长期缺 bootstrapper，BSD 尤其（`bootstrap/README.md:95-101` 记录 FreeBSD 换源自项目初期
就是目标之一，`:73-75` 里 BSD 的备选语言至今「待议」）。征集入口是 issue 230
（`README.md:295`、`README.md:334`），提 issue 前先搜一遍是否已有同类 PR。