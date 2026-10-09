# chsrc 二进制获取

在没有 chsrc 时如何获得它。本文覆盖包管理器、官方预编译、源码构建、bootstrapper 四条路径的决策。

## 决策树

```text
需要 chsrc
   ↓
有包管理器？（winget / scoop / brew / apt）
   ├─ 是 → 用包管理器（最省心，但版本可能落后于 pre tag）
   └─ 否 ↓
平台有官方预编译产物？
   ├─ 是 → 下载它（记录版本 + URL + SHA256）
   └─ 否（只有 BSD）↓
构建依赖齐了吗？（编译器 + -I 的头文件）
   ├─ 是 → 单文件编译，或用 just / make
   └─ 否 ↓
死锁 → 见 references/bootstrap.md
```

## 官方预编译产物

Release 托管在 Gitee：`https://gitee.com/RubyMetric/chsrc/releases/download/pre/<产物名>`
上游主仓在 GitHub，Gitee 是 README 明示的可用替代源。

| 平台 | 架构 | 产物名 |
| :-- | :-- | :-- |
| Windows | x64 | `chsrc-x64-windows.exe` |
| Windows | x86 | `chsrc-x86-windows.exe` |
| Windows | arm64 | `chsrc-arm64-windows.exe` |
| Linux | x64 | `chsrc-x64-linux` |
| Linux | aarch64 | `chsrc-aarch64-linux` |
| Linux | riscv64 | `chsrc-riscv64-linux` |
| Linux | armv7 | `chsrc-armv7-linux` |
| macOS | x64 | `chsrc-x64-macos` |
| macOS | arm64 / aarch64 | `chsrc-aarch64-macos` |

```bash
# Linux / macOS（下载后记得加执行权限）
curl -L https://gitee.com/RubyMetric/chsrc/releases/download/pre/chsrc-x64-linux -o chsrc
chmod +x ./chsrc
```

**BSD 没有任何预编译产物。** GitHub 官方 runner 不提供 BSD，上游也未发布 BSD 包。BSD 上
直接走源码构建；依赖本身也缺时走 bootstrapper。

## 包管理器

| 平台 | 命令 |
| :-- | :-- |
| Windows | `winget install RubyMetric.chsrc` |
| Windows | `scoop install chsrc` |
| macOS / Linux | `brew install chsrc` |
| Linux x64 | `apt install ./chsrc_latest-1_amd64.deb` |

包管理器装的版本可能落后于 `pre` tag。`pre` 通常比包管理器更新，适合修 Bug 后立即使用。

## 一键安装脚本

上游自带两个脚本，走 `tool/` 目录：

```powershell
# Windows
& { $(iwr -useb https://chsrc.run/windows) } -Version pre | iex
```

```bash
# macOS / Linux，非 root 时默认装到 ~/.local/bin
curl -L https://gitee.com/RubyMetric/chsrc/raw/main/tool/installer.sh | bash
```

若上方链接不可达，可改用 Gitee 上的原始文件地址 `https://gitee.com/RubyMetric/chsrc/raw/main/tool/installer.ps1`
（Windows）或 `.../tool/installer.sh`（macOS / Linux）。

### installer.ps1 的实际逻辑

`tool/installer.ps1` 的行为可以照抄，不必自己写探测逻辑：

| 步骤 | 位置 | 行为 |
| :-- | :-- | :-- |
| 架构探测 | `:149-161` | 读 CPU 架构：`0` → `x86`，`x64` → `x64`，`12` → `arm64`；其余值直接 `Output_Error` 报 `Unsupported architecture` |
| URL 拼接 | `:168-173` | `https://gitee.com/RubyMetric/chsrc/releases/download/{target_version}/chsrc-{arch}-windows.exe`；`target_version` 为 `pre` 时不加 `v` 前缀，指定版本时加 |
| 下载 | `:199` | `Invoke-WebRequest -OutFile ... -ErrorAction Stop` |
| 清理 | `:210-212` | 失败路径下 `Remove-Item -Recurse -Force` 删除临时目录 |

支持 `-Version` 指定版本。不指定时默认 `pre`。

## 源码构建

chsrc 是**单文件编译项目**：`src/chsrc-main.c` 用 `#include` 汇聚全部框架与 recipe 的 `.c`，
编译过程没有中间目标文件。因此下面这条命令**就是真实的构建方式**，不是野路子：

```bash
clang -Iinclude -Ilib src/chsrc-main.c -o chsrc
```

`README.md` §安装·BSD 直接给出了这条命令。`-Iinclude -Ilib` 是必需的——缺了会因找不到
`xy.h` 而编译失败。

构建模式通过宏区分：

| 模式 | 命令 | 额外宏 |
| :-- | :-- | :-- |
| DEV | `just b` / `make b` | 无 |
| DEBUG | `just bd` / `make bd` | `-DXY_DEBUG` |
| RELEASE | `just br` / `make br` | 优化 + 可选 `STATIC=1` 静态链接 |

`justfile` 用于原生 Windows；`make` 用于 Linux / macOS / MSYS2；**上游 CI 只走 make**，
justfile 里没有 CI 目标。`make bcir` 是 CI-RELEASE 模式（Linux 静态链接、macOS 固定 gcc-14），
justfile 里没有对应目标。

覆盖编译器：`just CC=clang bd`、`make CC=gcc br`。

### 构建工具怎么选

```text
有 just（原生 Windows）→ just b
有 make（Linux/macOS/MSYS2/CI）→ make b
两者都没有，但有 clang / gcc / cc → clang -Iinclude -Ilib src/chsrc-main.c -o chsrc
编译器也没有 → 见 references/bootstrap.md
```

## 记录来源

下载或安装完成后，记录版本、来源 URL、SHA256：

```json
{
  "chsrc_version": "0.2.7.2",
  "source_url": "https://gitee.com/RubyMetric/chsrc/releases/download/pre/chsrc-x64-linux",
  "sha256": "…",
  "fetched_at": "2026-10-06T00:00:00Z",
  "tag": "pre"
}
```

`pre` 是**移动 tag**——上游每次发布它都会变。做 A/B 对比评测时，两组必须共用同一份二进制
文件，不各自下载，否则差异会被误判成被测对象的差异。