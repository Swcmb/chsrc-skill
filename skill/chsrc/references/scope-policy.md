# 作用域策略与安全约束

作用域决定 chsrc 把源写到哪里。选错作用域的代价差异极大：`UserScope` 写用户配置目录，
删掉即恢复；`SystemScope` 写系统路径且需要 root，误写的清理成本高一个量级。

---

## 一、四种 Scope 值

`Scope_t` 定义在 `src/framework/struct.h:121-134`：

```c
#define NumberOfScopeType    3
typedef enum Scope_t {
  ProjectScope,
  UserScope,
  SystemScope,
  ImplementationDefinedScope,
} Scope_t;
```

| 值 | 含义 |
| :-- | :-- |
| `ProjectScope` | 项目级，仅当前项目目录 |
| `UserScope` | 用户级，当前用户全局 |
| `SystemScope` | 系统级，全系统可见，通常需要 root |
| `ImplementationDefinedScope` | dish 的默认作用域，由 dish 自身按实际情况决定 |

`ImplementationDefinedScope` 是默认初始值（`src/framework/base.c:51`）。源码注释说它
「不是一种新类型，而是表示根据实际情况决定的作用域」，换源完成后用户看到的作用域仍落在
三者之一（`src/framework/struct.h:116-119`、`src/framework/struct.h:128-132`）。

它在两种情形下产生实际差异：

- dish 三个作用域都实现了，`ImplementationDefinedScope` 直接落到 dish 的 `default_scope`。
- dish 的换源动作在某个作用域失败时退到另一个作用域，此时无法用单一值描述。
  `src/recipe/lang/Java.c:22-25` 注释即此例：先试 `SystemScope`，失败再试 `UserScope`。

框架不会把 `ImplementationDefinedScope` 解析成某个具体值再传给 recipe。recipe 直接调
`chsrc_in_user_scope_mode()` 等谓词判断，默认模式下三者全为 false，recipe 走自己的 else
分支（`src/framework/base.c:60-63`）。

## 二、ScopeCapability 与 scope_caps[3]

`ScopeCapability_t` 定义在 `src/framework/struct.h:140-147`，四个值：

| 值 | 含义 |
| :-- | :-- |
| `ScopeCap_Unknown` | 未知，缺乏对该 dish 的细致了解 |
| `ScopeCap_Unable` | 不支持该作用域 |
| `ScopeCap_Able_But_Not_Implemented` | 支持，chsrc 尚未实现 |
| `ScopeCap_Able_And_Implemented` | 支持且已实现 |

每个 dish 用 `scope_caps[3]` 逐个声明（`src/framework/struct.h:218`），下标常量是
`ScopeCap_Slot_Project` / `_User` / `_System`，值为 0 / 1 / 2（`src/framework/struct.h:136-138`）。
另有 `default_scope` 字段（`src/framework/struct.h:219`）。

声明有三条硬约束：

1. **索引由 `chef_set_scope_cap()` 固定映射，不直接取 enum 值**（`src/framework/chef.c:343-355`）。
   传入 `ImplementationDefinedScope` 会触发 `chsrc_breakdown()`（`src/framework/chef.c:356-359`）。
2. **`chef_set_default_scope()` 必须在 `chef_set_scope_cap()` 之后调用**（`src/framework/chef.c:366-367`）。
   默认作用域的能力若不是 `Able_And_Implemented`，直接 breakdown（`src/framework/chef.c:393-396`）。
3. **系统级 dish 走 `chef_set_os_scope()`**：Project 与 User 均置 `Unable`，System 置
   `Able_And_Implemented`，默认作用域 `SystemScope`（`src/framework/chef.c:404-413`）。
   全部 apt / yum / pacman / BSD 系发行版用它，共 30 个 recipe。

推论：发行版包管理器只能系统级换源。`chsrc set -scope=user ubuntu` 会被拒绝。

## 三、-scope 的两种写法

两个分支都在源码里处理（`src/chsrc-main.c:163-206`）：

```bash
chsrc set -scope=system apt     # 等号写法，值在同一个 argv 里
chsrc set -scope system apt     # 空格写法，值取 argv[i+1]
```

空格写法会额外把菜品位与镜像位各后移一位（`src/chsrc-main.c:169-170`），保证 dish 位置正确。

取值四个，大小写不敏感（`src/chsrc-main.c:176-191`）：

| 值 | 结果 |
| :-- | :-- |
| `project` | `ProjectScope` |
| `user` | `UserScope` |
| `system` | `SystemScope` |
| `default` | `ImplementationDefinedScope` |

其余取值报「无效的换源作用域」，退出码 `Exit_Unknown`（`src/chsrc-main.c:192-205`）。
提示文本给出的合法值列表也是这四个（`src/chsrc-main.c:197`、`src/chsrc-main.c:202`）。

`-default` 等价于省略 `-scope`，因为默认值就是 `ImplementationDefinedScope`。

选项必须写在 dish 名之前。解析器每遇到一个 `-` 开头参数就把菜品位与镜像位各后移一位
（`src/chsrc-main.c:237-238`）。

## 四、-local 已弃用

`-local` 仍然生效，映射到 `ProjectScope`，但会先打一条 warning 再继续
（`src/chsrc-main.c:156-162`）：

```
-local 选项已弃用，请使用 -scope=project
```

使用 `-scope=project` 替代。

## 五、能力检查只在 set 发生

`chsrc_check_scope_capability()`（`src/framework/source.c:796-844`）逐项比对
`scope_caps`，只要不等于 `Able_And_Implemented` 就报错退出，退出码 `Exit_UserCause`。
报错文案是「不支持<级别>换源，请使用 chsrc ls <dish> 查看支持的作用域以及默认作用域」。

关键点：

- **只在 `set` 路径调用**（`src/framework/waiter.c:870`）。`reset` 直接调 `resetfn`
  （`src/framework/waiter.c:901-903`），不做能力检查。
- **仅在显式指定 `-scope` 时检查**。三个 `chsrc_in_*_scope_mode()` 谓词全为 false 时
  函数体不进入任何分支（`src/framework/source.c:808-843`），默认模式不做检查。
- **combo dish 跳过检查**，交给各 sub dish 逐个检查（`src/framework/waiter.c:863-867`）。

## 六、用 chsrc list 发现能力

`chsrc list <dish>` 会调用 `cli_print_dish_features()`（`src/framework/waiter.c:194`），
逐作用域打印能力，并附带可直接复制的命令（`src/framework/waiter.c:225-294`）：

```text
 项目级换源 | chsrc set -scope=project pip  (不支持)
 用户级换源 | chsrc set -scope=user    pip  (支持且已实现)
 系统级换源 | chsrc set -scope=system  pip  (不支持)
 = 默认作用域 | chsrc set -scope=default pip (= 用户级)
```

标记含义：绿色对勾 = `Able_And_Implemented`，红色叉 = `Unable`，黄半对勾 =
`Able_But_Not_Implemented`，红色问号 = `Unknown`（`src/framework/waiter.c:250-271`）。

默认作用域若为 `ImplementationDefinedScope`，这一行显示「由实现定义」
（`src/framework/waiter.c:286-288`）。

**定作用域前先跑 `chsrc list <dish>`，比猜省事。** 命令行直接照抄输出里的写法即可。

## 七、-dry 是全局闸门

`-dry` 置 `ProgMode.DryRunMode`（`src/chsrc-main.c:211-214`），进入时打印一条提示
（`src/chsrc-main.c:242-249`）：「开启Dry Run模式，模拟换源过程(跳过测速)，命令仅打印并不运行」。

守卫位置已逐处核实，全部在任何副作用之前：

| 位置 | 守卫对象 | 拦截的动作 |
| :-- | :-- | :-- |
| `src/framework/OS.c:324` | `chsrc_run()` | `system(cmd)` 在 `:329`，先于它返回 |
| `src/framework/OS.c:382` | `chsrc_run_directly()` | `system(cmd)` 在 `:386`，先于它返回 |
| `src/framework/OS.c:750` | `chsrc_append_to_file()` | `fopen(..., "a")` 在 `:759` |
| `src/framework/OS.c:805` | `chsrc_prepend_to_file()` | `fopen(..., "w")` 在 `:815` |
| `src/framework/OS.c:836` | `chsrc_overwrite_file()` | `fopen(..., "w")` 在 `:845` |
| `src/framework/OS.c:875` | `chsrc_backup()` | 备份命令执行 |

四个文件类守卫用 `goto log_anyway`，跳到打印逻辑，所以 `-dry` 下仍会打印将要写入的
文件名与内容提示，只是不落盘。

测速两处：

- `src/framework/source.c:355` —— 默认跳过测速横幅。
- `src/framework/source.c:370-374` —— 直接 `return sources[1]`。

**读 `-dry` 输出时注意**：`sources[0]` 是上游默认源，reset 走它
（`src/framework/source.c:352-353`）。`-dry` 下返回的 `sources[1]` 是源列表里的第二个，
是一个占位值，与真实测速结果无关。预览里出现的 URL 不代表最终会被选中。

### -dry 的已知缺口

`ensure_debian_or_ubuntu_old_sourcelist()`（`src/recipe/os/APT/common.h:58`）内部用裸
`fopen(OS_Apt_SourceList, "w")` 直接写 `/etc/apt/sources.list`
（`src/recipe/os/APT/common.h:117`），不经 `chsrc_overwrite_file()`，没有 dry 守卫。
调用点在 `src/recipe/os/APT/Ubuntu.c:127`、`src/recipe/os/APT/Debian.c:127`、
`src/recipe/os/APT/Debian.c:145`。

触发条件是 `/etc/apt/sources.list` 不存在（`src/recipe/os/APT/common.h:60`）。

**别把它套到所有 Debian 机器上。** `os_debian_setsrc()` 在 `src/recipe/os/APT/Debian.c:117-122`
先检查 DEB822 源文件，存在就走 `os_debian_setsrc_for_deb822()` 并**直接 return**，根本到不了
这个函数：

```c
if (chsrc_check_file (OS_Debian_SourceList_DEB822))
  {
    chsrc_note2 ("将基于新格式(DEB822)换源");
    os_debian_setsrc_for_deb822 (option);
    return;                    // ← 提前返回，缺口不触发
  }
```

所以实际会踩到的场景是：

| 场景 | 是否触发 |
| :-- | :-: |
| Debian 12 / Ubuntu 24.04 全新安装（有 deb822 文件） | 否，提前 return |
| 仍在用旧格式 `/etc/apt/sources.list` 的老系统 | 是 |
| 精简 Docker 镜像，两者都没有 | 是 |
| 旧配置里有 CDROM 源，触发删除-重建路径（`Debian.c:145` 二次调用） | 是 |

在只读挂载、容器、chroot 等隔离环境里对上述第二类场景做 debian/ubuntu 的 `-dry` 演练时，
这一步仍可能写盘。把 `-dry` 当作尽力而为的手段，隔离手段仍需另外准备。

## 八、SystemScope 的实际写面

系统级 dish 写的都是系统路径。已核实的例子：

- apt 系：`/etc/apt/sources.list`、`/etc/apt/sources.list.d/`
  （`src/recipe/os/APT/common.h:15-16`、`src/recipe/os/APT/common.h:25-26`）
- MSYS2/pacman 系：`/etc/pacman.d/mirrorlist.*`
  （`src/recipe/os/pacman/MSYS2.c:46-51`，路径按 MSYS2 root 解析）

**chsrc 自己不做提权。** `chsrc_ensure_root()`（`src/framework/OS.c:280-301`）只检测
`id -u`，非 root 时打印「请在命令前使用 sudo 或切换为root用户来保证必要的权限」
并以 `Exit_UserCause` 退出。系统级换源要用户自己在命令前加 `sudo`。

Windows 系统级写入的具体落点，在当前源码中未核实到：`src/` 下 grep 不到
`ProgramData`、`reg add`、`HKEY_LOCAL_MACHINE` 之类的写入路径，chsrc 的 Windows 侧写操作
主要委托给厂商命令行：

- winget：`chsrc_run("winget source add ...")`
  （`src/recipe/ware/WinGet.c:53-56`），写点由 winget 自己决定。
- npm：`chsrc_run("npm config set registry ...")`（`src/recipe/lang/JavaScript/NPM.c:57-62`），
  项目级用 `--location project`，其余走默认位置（`src/recipe/lang/JavaScript/NPM.c:57-60`）。
- pip：`python -m pip config --user set global.index-url`
  （`src/recipe/lang/Python/Pip.c:166`）。

这些委托命令的落盘位置在 chsrc 控制范围之外，且大多没有 undo。Windows 上换源前用
`-dry` 逐条读将要执行的命令，比依赖隔离更可靠。

## 九、HOME 隔离的边界

`xy_os_home()`（`lib/xy.h:1271-1281`）在 Windows 上返回 `%USERPROFILE%`，其它平台返回
`$HOME`。两者读的是不同环境变量。

据此可以确定：

- **Linux/macOS 上覆盖 `HOME` 能隔离**依赖 HOME 的 dish。`rustup`、`dart pub`、
  `flutter`、`homebrew` 追加的是 `xy_bashrc` / `xy_zshrc`，即字面量 `~/.bashrc` /
  `~/.zshrc`（`lib/xy.h:1358-1359`）。
- **Windows 上覆盖 `HOME` 对 chsrc 无效**，`xy_os_home()` 只读 `%USERPROFILE%`。
- **Windows 上还有绕开 HOME 的路径**。uv 的用户级配置取 `%APPDATA%`
  （`src/recipe/lang/Python/uv/common.h:58`）；pip 的用户级写入实测落在
  `C:\Users\<用户>\AppData\Roaming\pip\pip.ini`，源码注释记录了这一点
  （`src/recipe/lang/Python/Pip.c:165`）。两者都在 `HOME` 之外。

结论：临时 HOME 只能作为 Linux/macOS 上的额外一层。Windows 上的隔离手段是 `-dry`，
或者在虚拟机 / 容器里跑完整流程。

## 十、作用域选择规则

按顺序判断：

1. **先跑 `chsrc list <dish>`**（`chsrc list <dish>`），读出该 dish 支持的作用域与默认作用域。
2. **默认取 `-scope=user`。** 用户说「换个源」多数指用户级全局。pip、npm、cargo 等
   language 级 dish 的 `default_scope` 就是 `UserScope`。
3. **发行版包管理器只能用系统级**，它们的 `default_scope` 是 `SystemScope`，
   且 `-scope=user` 会被 `chsrc_check_scope_capability()` 拒掉。此时需要用户确认
   在命令前加 `sudo`，并说明会写 `/etc/...`。
4. **升到 `SystemScope` 前确认用户意图并说明影响面**：影响全系统所有用户，
   清理需要 root，且没有内建 undo。
5. **组合 dish（`py_group` / `js_group` / `uv`）逐个子菜品确认**，子菜品各自的作用域
   能力可能不同。

## 十一、每次写操作的标准序列

```bash
chsrc list pip                     # 1. 查支持的作用域与默认值
chsrc set -dry -scope=user pip     # 2. 预览：选项写在 dish 之前
chsrc set -scope=user pip          # 3. 执行
chsrc get pip                      # 4. 复查当前源
```

作用域在预览之前就要定下来。`-dry` 与实际执行使用同一个 `-scope`，预览到的作用域才
与真实写入一致。

`get` 是只读操作，任何时候都可以跑。改完不跑 `get` 无法确认实际落盘的源，
原因可能是测速选了另一个源、写入被权限拒绝、或配置被其它工具覆盖。