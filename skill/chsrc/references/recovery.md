# 故障恢复与回滚

## 回滚换源

```bash
chsrc reset -dry pip      # 先看会改什么
chsrc reset pip           # 恢复默认源
chsrc get pip             # 确认
```

`reset` 把配置恢复成上游默认源。它是写操作，同样先 `-dry`。

## 退出码

| 码 | 宏 | 含义 |
| --: | :-- | :-- |
| 0 | `Exit_OK` | 成功 |
| 1 | `Exit_Fatal` | 致命错误 |
| 2 | `Exit_Unknown` | 未知菜品或参数 |
| 3 | `Exit_Unsupported` | 不支持的操作 |
| 4 | `Exit_UserCause` | **用户侧问题**，通常是缺依赖 |
| 5 | `Exit_MaintainerCause` | recipe 自身的问题 |
| 6 | `Exit_ExternalError` | 外部错误（网络、命令失败） |

**退出码 4 是最常见的失败码。** 它由 `chsrc_ensure_program`（`src/framework/OS.c:244`）、
`chsrc_ensure_root`（`OS.c:280`）以及 dish 的 prepare 阶段（`src/framework/dish.c:294,306`）
触发——即「要继续就必须先有某个东西，而那个东西没有」。

具体触发点：

| 缺什么 | 表现 | 处理 |
| :-- | :-- | :-- |
| `curl` | 测速无法进行 | 装 curl；或用 `chsrc set <dish> <mirrorCode>` 直接指定镜像站，跳过测速 |
| 目标程序本身（如没装 npm 却要 `chsrc set npm`） | dish prepare 失败 | 先装目标程序 |
| root 权限 | `-scope=system` 被拒 | 换 `-scope=user`，或用 sudo |
| 网络不通 | 退出码 6 | 代理、防火墙、镜像站可达性 |

**绕过测速**是应对 curl 缺失的实用手段：`chsrc set <dish> tuna` 给出指定镜像站时不需要测速。

### `No module named pip`：多个 Python 指向不同解释器

这是 chsrc 换 pip 时最常见的失败，报错来自 chsrc 实际调用的 `python3`：

```text
D:/ProgramFiles/msys64/ucrt64/bin/python3.exe: No module named pip
[chsrc 检查] ✓ 命令 python3 存在
```

chsrc 的 dish 只检查「`python3` 这个命令存不存在」，存在就继续往下走。机器上有多个 Python
时（Windows 上尤其常见：MSYS2/Ubuntu WSL 的 `python3`、Anaconda 的 `python`、系统自带的
`python`），**存在的那个 `python3` 未必是你日常用的那个解释器**。

关键点：**每个解释器有自己独立的一份 pip 配置**。MSYS2 的 pip 和 Anaconda 的 pip 读的是
两个不同的 `pip.ini`，改一个不影响另一个。chsrc 调的是 `python3` 那一份，你 `pip install`
用的是 `python` 那一份——两边看到的源可能一直不一样。

排查顺序：

```text
1. 确认 chsrc 调的 python3 是哪个
   chsrc set -dry pip        # 看它打算执行哪条命令
2. 确认你日常用的是哪个
   python -c "import sys; print(sys.executable)"
   python -m pip --version   # 这才是你实际在用的 pip
3. 两个路径不同 → 两份配置，用 python -m pip 改你真正在用的那份
   python -m pip config --user set global.index-url <URL>
   python -m pip config get global.index-url
4. 复查也用同一个解释器
   python -m pip config list
```

chsrc 的 `get`/`set` 失败时，不要改用「手动编辑 pip.ini」绕过去——那会改到 chsrc 管的
那份，而你的 pip 读的是另一份。统一用 `python -m pip` 走你实际在用的解释器。

## 换源后没生效的排查顺序

```text
① chsrc get <dish>                    实际写到了哪个文件？源地址是什么？
② 检查配置文件是否被其他工具覆盖        npm config、conda config、pip config 都会写同一批文件
③ 检查作用域是否选错                    user scope 与 system scope 落在不同位置
④ 检查写的文件在当前平台是否被读取        例如 pip 只读 pip.conf 与 pip.ini，平台不同路径不同
⑤ 检查选项顺序                          chsrc get pip -no-color 会把选项当菜品名
```

## 配置被其他工具覆盖

pip / npm / conda 都有自己的配置命令，改的是同一批文件：

```text
npm config set registry ...
pip config set global.index-url ...
conda config --set channels ...
```

这类工具在 chsrc 之后运行时，会覆盖 chsrc 的结果。判断归属：

| 场景 | 选谁 |
| :-- | :-- |
| 用户想统一管理所有换源 | chsrc |
| 用户只管这一个工具，且要精细控制（如代理、认证） | 该工具自己的配置命令 |
| 用户在 CI 脚本里 | 该工具自己的配置命令——不假设 chsrc 存在 |

反复用 chsrc `set` 覆盖不属于自己的配置是无解的循环。直接用对应工具的配置命令。

## 备份与损坏

chsrc 在改文件前会备份：备份路径是原文件路径加 `.bak` 后缀（`src/framework/OS.c:35`）。
备份按平台选不同命令——BSD 与 macOS 的 `cp` 不支持 `--backup`，busybox 的 `cp` 也不支持，
因此有多条分支（`OS.c:908-937`）。文件不存在时跳过备份，不报错。

恢复：

```bash
cp <原路径>.bak <原路径>
```

chsrc 自身的 CLAUDE 约定不写用户数据文件，但备份文件是明确的例外。

## Windows 特有的坑

| 坑 | 表现 | 处理 |
| :-- | :-- | :-- |
| `%USERPROFILE%` 与 `%APPDATA%` 不同 | chsrc 的家目录是 `USERPROFILE`（`lib/xy.h:1276-1277`），下游工具的配置常在 `APPDATA`。写错位置 chsrc 自己 get 不到 | 先 `chsrc get <dish>` 确认 chsrc 认为的落点 |
| system scope 委托给厂商命令行 | `winget source add` 之类会改系统级源配置，`chsrc reset` 不一定能撤回 | 优先用 `-scope=user`；系统级改前确认来源 |
| `HOME` 隔离无效 | 设了临时 `HOME` 但配置照样被改 | 见下 |
| 两份配置并存 | 优先级不明确，行为随环境变 | 清理掉重复的那份 |

**`HOME` 隔离对 chsrc 无效。** chsrc 的家目录在 Windows 上取 `USERPROFILE`
（`lib/xy.h:1276-1277`），在 Linux/macOS 上取 `HOME`，两者都不受 `HOME` 覆盖影响。
Windows 上的隔离手段是 `-dry`。

## 什么时候停下来问

以下情形停止尝试并向用户确认：

- 同一操作连试三次结果不一致
- 需要 `system` 作用域但用户未确认过影响面
- 要改的文件不属于本次任务范围
- 诊断信息不足以判断根因，且继续试会产生不可逆改动

## 恢复后的验证

```bash
chsrc get <dish>     # 确认源已回到预期状态
```

恢复也是写操作，同样要复查。