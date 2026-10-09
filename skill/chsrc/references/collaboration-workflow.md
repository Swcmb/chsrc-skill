# 与 chsrc 上游协作

上游仓库：`RubyMetric/chsrc`（GitHub 主仓 + Gitee 镜像）。本文覆盖什么时候该向上游反馈、
怎么提，以及提之前该做什么准备。

## 分支模型

| 分支 | 定位 |
| :-- | :-- |
| `dev` | 开发分支，所有 PR 的目标分支 |
| `main` | stable，保证可编译可运行 |
| `gh-build` | 发版时由维护者推送 |

**提交 PR 时务必把 PR 指向原仓库的 `dev` 分支**，不是 `main`。

## 什么时候该向上游提 issue

该提：

- 某个 dish 换源失败（配置写不进去、格式不对、权限被拒）
- 某个镜像站失效（URL 打不开、被墙、证书过期）
- 某个平台不被支持（尤其是非官方 runner 的平台，如 BSD）
- 文档错误（命令写错、选项写错）
- 某个 dish 缺少 `-scope=` 能力（实际支持但 `scope_caps` 声明为 `Unable`）

不该提：

- 本地环境问题（编译器没装、权限配置错、代理故障）
- 用户自己配置写错了
- 想要一个上游没有的私有源
- 用法问题——先读 `doc/` 下的文档和 `chsrc help`

## 提之前必做

```text
1. 搜索同类 issue。issue 可能是 closed 的，或者已有 PR 在处理。
2. 用最新版复现。旧版本的问题可能已修。
3. 确认不是本地环境问题。
4. 收集诊断信息（见下）。
```

## 诊断信息清单

上游无法复现就无法修。issue 或 PR 里必须写全：

```text
平台与版本      Windows 11 / macOS 15 / Debian 12 / FreeBSD 14 …
架构            x64 / arm64 / aarch64 …
chsrc 版本      chsrc --version 的输出
完整命令        一字不改地贴出来，包括选项顺序
完整输出        报错信息原文，不要转述
预期行为        原本应该发生什么
实际行为        实际发生了什么
是否有 -dry 输出 chsrc set -dry <dish> 的输出往往最关键
```

选项顺序特别值得强调：`chsrc get pip -no-color` 会把 `-no-color` 当成菜品名，
报「暂不支持的换源菜品」。复现前先确认选项写在菜品名之前。

## 提 PR

```text
1. fork 原仓库
2. 从 fork 的 dev 分支切出特性分支
3. 改动 + 自测（DEBUG 构建跑一遍受影响的 dish）
4. 提交到 fork
5. 开 PR 指向原仓库的 dev 分支
6. PR 描述里写清楚：改了什么、为什么、怎么验证的
```

原生 Windows 用 `just`，Linux / macOS / MSYS2 / CI 用 `make`。上游 CI 只走 make。

## 给上游加一个新 dish

从用户视角看，一个 dish 要能被调用，必须**登记到菜单里**，否则用户根本调不到它——
`chsrc get <dish>` 会报「暂不支持的换源菜品」。

登记涉及三处（`src/recipe/menu.c`）：

| 段 | 宏定义 | 登记范围 |
| :-- | :-- | :-- |
| 镜像站 | `:122` | `xy_seq_push(ProgStore.mirror_sites, &t)` |
| `pl_*` dish | `:166` | `xy_seq_push(ProgStore.pl, &pl_##t##_dish)` + 挂 `preparefn` |
| `os_*` dish | `:203` | 同上，写 `ProgStore.os` |
| `wr_*` dish | `:237` | 同上，写 `ProgStore.wr` |

段与段之间用 `#undef add` 分隔（`:162` / `:201` / `:235` / `:248`）。加新 dish 时注意
`pl_` / `os_` / `wr_` 前缀对应目录 `lang/` / `os/` / `ware/`。

写 recipe 的完整步骤见上游 `doc/10-如何编写recipe.md`。

## 报告 bootstrapper

上游在 `bootstrap/README.md` 里征集各平台的 bootstrapper（issue #230 是 BSD 的入口）。
写好一个 bootstrapper 值得提 PR——这解决了所有同类用户的死锁。

写法见 `references/bootstrap-authoring.md`。

## 镜像站失效的处理

镜像站失效是时效性问题，容易被误判成「chsrc 坏了」。区分方法：

```text
chsrc ls                     列出全部镜像站及其 code
chsrc measure <dish>         测速——被移除的站会显示为不可用
chsrc get <dish>             看当前实际指向哪里
```

若某个镜像站确实长期失效，上游通常会更新或标记。提 issue 时给出该站 URL 与失效现象。