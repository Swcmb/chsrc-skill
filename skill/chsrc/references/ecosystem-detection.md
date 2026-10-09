# 用户说法 → dish 映射

用户描述的是语言、用途或生态，chsrc 的菜名是工具名。本篇给出两者的对照表，以及查不到时的处理顺序。

---

## 一、dish 名是裸名

命令行里传的 dish 名就是源码里 `def_dish(...)` / `def_combo_dish(...)` 的第二个参数，
形如 `pip`、`ubuntu`、`rust_cargo`。

`pl_` / `os_` / `wr_` 是 C 变量名前缀（`pl_pip_dish`、`os_ubuntu_dish`），只出现在
`src/recipe/` 与 `src/framework/` 的源码里。传给 CLI 的任何位置都不带这三个前缀。

| 源码里的变量 | 传给 CLI 的 dish 名 |
| :-- | :-- |
| `pl_pip_dish` | `pip` |
| `os_ubuntu_dish` | `ubuntu` |
| `wr_homebrew_dish` | `homebrew` |

---

## 二、匹配规则

三条规则决定一个输入能否命中 dish：

1. **精确匹配，忽略大小写。** 比较函数是 `xy_streql_ic()`（`lib/xy.h:635`），
   逐字符比对、长度必须相同、大小写不敏感（`src/framework/waiter.c:490-494`）。
   `PIP`、`Pip`、`pip` 都能命中 `pip`；`pi`、`pips` 命中不了。
2. **每个 dish 有多个别名。** 别名串用 `/` 分隔，匹配时逐个拆开比较
   （`src/framework/waiter.c:56-85`）。`ruby` 的别名是
   `gem/ruby/rb/rubygem/rubygems/bundler`，所以这六个词都命中同一个 dish。
3. **三组按固定顺序查找：pl → os → wr。** 第一个命中的组立即返回
   （`src/framework/waiter.c:602-604`）。需要排除某个别名被前面一组抢走时，
   查这一步的顺序。

命中后只调用该 dish 自己的 `preparefn()` 填源信息，其余 recipe 全部跳过
（`src/framework/waiter.c:608-610`）。

---

## 三、完整 dish 清单（71 个）

`src/recipe/menu.c` 的 `chsrc_init_menu()` 用三个宏各注册一组：
pl 组 30 个（`menu.c:166-201`）、os 组 31 个（`menu.c:203-235`）、wr 组 10 个
（`menu.c:237-248`）。

### 3.1 pl 组（30 个）

| dish | 别名 | 口语说法 |
| :-- | :-- | :-- |
| `py_group` | `py` `python` | Python 整套工具链（套餐） |
| `pip` | `pip` | python 换源、Python 包加速、PyPI |
| `uv` | `uv` | uv 整套（套餐） |
| `uv_pypi_index` | `uv-pypi-index` | uv 下载包加速 |
| `uv_python_build` | `uv-python-build` | uv 下载 Python 解释器加速 |
| `js_group` | `js` `javascript` `node` `nodejs` | Node 整套（套餐） |
| `poetry` `pdm` `rye` `npm` `pnpm` `yarn` `bun` `nvm` `julia` | 各 dish 同名 | 各自独立的工具，见第四节 |
| `ruby` | `gem` `ruby` `rb` `rubygem` `rubygems` `bundler` | Ruby、Gem、Bundler |
| `go` | `go` `golang` `goproxy` | Go modules 代理 |
| `rust_cargo` | `rust` `cargo` `crate` `crates` | Rust 依赖、cargo |
| `rust_rustup` | `rustup` | Rust 工具链本身 |
| `java` | `java` `maven` `mvn` `maven-daemon` `mvnd` `gradle` | Maven、Gradle |
| `perl` | `perl` `cpan` | Perl、CPAN |
| `php` | `php` `composer` | PHP、Composer |
| `lua` | `lua` `luarocks` | LuaRocks |
| `dart` | `dart` `pub` | Dart、pub |
| `dart_flutter` | `flutter` | Flutter |
| `nuget` | `nuget` `net` `.net` `dotnet` | .NET、NuGet |
| `haskell` | `haskell` `cabal` `stack` `hackage` | Cabal、Hackage |
| `ocaml` | `ocaml` `opam` | opam |
| `r` | `r` `cran` | CRAN |
| `clojure` | `clojure` `clojars` `cloj` `lein` | Clojure、Leiningen |

别名出处：每个 recipe 首行的 `def_dish()`（例如 `src/recipe/lang/Ruby/Ruby.c:12`、
`src/recipe/lang/Java.c:5`）；三个套餐见第五节。

### 3.2 os 组（31 个）

按包管理器归组，同一行的 dish 共用一套包管理器与配置文件位置：

| 包管理器 | dish 名 |
| :-- | :-- |
| apt | `ubuntu` `debian` `linuxmint` `kali` `raspberrypi` `trisquel` `linuxlite` `armbian` `ros` `termux` `openkylin` `deepin` |
| dnf | `fedora` `almalinux` `rockylinux` `openeuler` `anolis` |
| pacman | `arch` `archlinuxcn` `manjaro` `omarchy` `msys2` |
| apk | `alpine` |
| portage | `gentoo` |
| xbps | `voidlinux` |
| eopkg | `solus` |
| zypper | `opensuse` |
| opkg | `openwrt` |
| pkg / pkgin | `freebsd` / `netbsd` / `openbsd` |

非直观别名：`arch`→`archlinux`、`archlinuxcn`→`archcn`、`raspberrypi`→`raspi`、
`rockylinux`→`rocky`、`almalinux`→`alma`、`msys2`→`msys`、`anolis`→`openanolis`、
`opensuse`→`opensuse-leap` / `opensuse-tumbleweed`、`openwrt`→`opkg` / `LEDE`。

系统包管理器的配置写在 `/etc` 下，作用域影响面大。执行前按
`references/scope-policy.md` 的流程处理。

### 3.3 wr 组（10 个）

| dish | 别名 | 口语说法 |
| :-- | :-- | :-- |
| `winget` | `winget` | Windows 应用装 |
| `homebrew` | `brew` `homebrew` | macOS、Homebrew、brew 换源 |
| `cocoapods` | `cocoa` `cocoapods` `cocoapod` | iOS 依赖 |
| `docker` | `docker` `dockerhub` | Docker Hub 拉镜像慢 |
| `flatpak` | `flatpak` `flathub` | Flatpak、Flathub |
| `nix` `guix` | 各 dish 同名 | Nix、Guix |
| `emacs` | `emacs` `elpa` | ELPA |
| `tex` | `latex` `ctan` `tex` `texlive` `miktex` `tlmgr` `mpm` | LaTeX、TeX Live |
| `anaconda` | `conda` `anaconda` | conda、Anaconda |

---

## 四、口语映射要点

### 4.1 「python 换源」→ `pip`

用户说「python」时按 `pip` 处理。字面输入 `python` 在 CLI 里命中的是套餐
`py_group`（别名含 `python`，`src/recipe/lang/Python/Python.c:5`），会展开成
pip + pdm + poetry 三个子菜品。把口语映射成 `pip` 才能命中单一目标。

用户明确说「Python 全部」「整套」「py」「所有 Python 工具」时才用 `py_group`。

### 4.2 「前端」→ npm / pnpm / yarn / bun / nvm

「前端」本身覆盖多个 dish，先确认具体工具。用户只说「前端」时默认 `npm`。
用户要「Node 全部」「JS 整套」时用 `js_group`；它的子菜品是 npm + yarn + pnpm，
不含 bun 与 nvm（`src/recipe/lang/JavaScript/JavaScript.c:15`）。

### 4.3 「系统包」/「apt」/「yum」→ 先定发行版

用户说包管理器名时先换成发行版，发行版才是 chsrc 的 dish：

| 用户说 | 追问 | 得到 |
| :-- | :-- | :-- |
| apt、apt 源 | Ubuntu 还是 Debian 系 | `ubuntu` / `debian` |
| yum、dnf | 哪个 RHEL 系发行版 | `fedora` / `rockylinux` / `almalinux` |
| pacman | Arch 系哪个 | `arch` / `manjaro` / `msys2` |

系统包管理器的源写在 `/etc` 下，作用域影响面大。执行前按
`references/scope-policy.md` 的流程处理。

### 4.4 「包管理器」有歧义 → 先问生态

「包管理器」同时指向语言级（pip、npm、cargo、gem、composer、nuget）和系统级
（winget、brew、apt、dnf、pacman）两类，且系统级还随发行版变化。听到这个词先问
一句是哪个生态，再落到 dish。

### 4.5 语言生态的组合说法

| 用户说 | dish |
| :-- | :-- |
| Go 代理、GOPROXY | `go` |
| Rust 装依赖慢（cargo 拉 crates） | `rust_cargo` |
| Rust 装工具链慢（rustup 拉发行版） | `rust_rustup` |
| Maven / Gradle / mvn | `java` |
| Composer | `php` |
| Bundler | `ruby` |
| conda | `anaconda` |
| Docker Hub 慢 | `docker` |

### 4.6 容器场景：先分清是哪一侧

用户说「Docker 里换源」时，歧义有三层。**先问清是哪一层，再谈 dish**：

| 用户说的 | 指的是 | 先确认什么 |
| :-- | :-- | :-- |
| 「Docker Desktop 换源」 | 宿主机 Docker 守护进程的 registry 配置 | 宿主机上有没有 chsrc |
| 「容器里换源」 | 容器**内部**的包管理器配置 | **容器内有没有 chsrc**——最小镜像里通常没有 |
| 「拉镜像慢」 | 走哪个 registry 拉镜像 | 是 Docker Hub 慢，还是私有 registry 慢 |

三条常被搞混的边界：

1. **容器内与宿主机是两套环境。** 容器里 `chsrc set pip` 只改容器文件系统，容器一删就没了；
   宿主机上装了 chsrc 不代表容器里能用。需要持久化就写进 Dockerfile，或挂载配置文件。
2. **容器里通常没有 chsrc。** 精简镜像（`alpine`、`distroless`、`python:slim`）不含 chsrc，
   也没有预编译产物可直接下载。此时先装 chsrc，或直接用该生态自己的配置方式。
3. **`docker` dish 只管 registry 层。** 它配置的是「从 Docker Hub 拉镜像走哪个镜像站」，
   不管容器内部 `apt` / `pip` / `npm` 的源。`ghcr.io`、`quay.io`、私有仓库各走自己的路径，
   这条 mirror 不参与。

---

## 五、套餐 dish（combo）

`py_group`、`js_group`、`uv` 三个是套餐。构造它们的宏 `chef_prep_this_combo_dish()`
只置 `prepared` 标志，展开的 `chef_allow_NOOP(t)` 是空宏
（`src/framework/chef.c:27`、`src/framework/chef.c:33`）。三者自身没有
`getfn` / `setfn` / `resetfn`，也没有自己的源列表。

| 套餐 | dish 名 | 子菜品 |
| :-- | :-- | :-- |
| Python | `py_group`（`py` `python`） | `pip`、`pdm`、`poetry` |
| Node | `js_group`（`js` `javascript` `node` `nodejs`） | `npm`、`yarn`、`pnpm` |
| uv | `uv` | `uv_pypi_index`、`uv_python_build` |

子菜品定义出处：`src/recipe/lang/Python/Python.c:15`、
`src/recipe/lang/JavaScript/JavaScript.c:15`、`src/recipe/lang/Python/uv/uv.c:15`。

对套餐执行 `get` / `set` / `reset` / `list` 时，chsrc 先打印子菜品构成，再对每个
子菜品调用同一个处理函数逐个操作（`src/framework/waiter.c:775`、
`src/framework/waiter.c:817`、`src/framework/waiter.c:888`、
`src/framework/waiter.c:634`）。展开由 `dish_has_sub_dishes()` 判定，条件是子菜品
数量大于 1（`src/framework/dish.c:37`）。

执行层面按「多个独立写操作」对待：先 `-dry` 预览全部子菜品，再执行，最后
`get` 复查每个子菜品。`set py_group tuna` 一次改动的是 pip、pdm、poetry 三份配置。

`py_group` 与 `js_group` 额外声明了「所有子菜品共用同一份源」
（`chef_all_sub_dishes_use_same_source_from`，`src/framework/chef.c:582`），共享源分别取自
`pl_pypi_dish`（`Python.c:17`）与 `pl_npm_dish`（`JavaScript.c:16`）。
指定镜像站时，chsrc 会先确认该镜像站是否为至少一个子菜品提供源，否则报
「镜像站存在，但未提供该软件源」（`src/framework/dish.c:289-294`）。

---

## 六、菜单之外的内部 dish

有两个 dish 只供其他 recipe 复用，别名被强制写成 `__internal__sources_only_dish__`
包裹的形式（`src/framework/chef.c:19`），CLI 输入匹配不上：

| 变量 | 内部别名 | 用途 |
| :-- | :-- | :-- |
| `pl_pypi_dish` | `__internal__sources_only_dish__pypi__` | `py_group` 的共享源 |
| `pl_nodejs_binary_dish` | `__internal__sources_only_dish__nodejs-binary__` | Node 二进制源 |

两者的 `preparefn` 由 `chef_set_preparefn_for_sources_only_dish()` 单独挂载，
不经 `menu.c` 的 `add()` 注册（`src/recipe/menu.c:250-251`、`src/framework/chef.c:36`）。

用户口语里的「pypi」「PyPI 源」映射到 `pip`；「Node 二进制」「nodejs 下载」对应
`nvm` 或 `js_group`，由 `js_group` 展开处理。

---

## 七、映射不到时

按顺序处理：先 `chsrc ls` 列出全部镜像站 code（`src/framework/waiter.c:40`）排除
镜像站侧拼写问题；再 `chsrc list <guess>` 试候选 dish，退出码 2 表示未知菜品
（`src/framework/waiter.c:40` 附近同族输出）。检查输入没带 `pl_`/`os_`/`wr_` 前缀，
并确认拼写——`archlinuxcn`、`linuxmint`、`rockylinux`、`voidlinux`、`dart_flutter`、
`uv_pypi_index`、`rust_cargo` 都是无连字符裸名。仍无匹配时向用户确认工具名。

私有源与公司内网源不在 chsrc 覆盖范围内，走各自的配置文件处理。