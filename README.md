# chsrc-skill

让 LLM 正确使用 [chsrc](https://github.com/RubyMetric/chsrc) 换源的 Agent Skill。

> [!TIP]
> **换源不是 AI 能即兴推断的事。** 该不该用 chsrc、装在哪、换哪个作用域、换完怎么复查——这些取决于 chsrc 上游的具体实现：命令集只有 5 个动作、选项必须写在菜品名之前、`-scope` 有空格和等号两种写法、系统级写入在 Windows 上落 `C:\ProgramData`。本 Skill 把这些固化成 [`skill/chsrc/SKILL.md`](skill/chsrc/SKILL.md)，细节下沉到 10 篇 `references/`。

> [!NOTE]
> 本 Skill 面向 **Claude Code 与任何支持 Agent Skills 格式的环境**。内容全部对照 chsrc v0.2.7.2 源码核实，每条命令都能回溯到具体行号。

## 📦 包含什么

```
skill/chsrc/
├── SKILL.md                    # 入口：触发判定、执行流程、命令参考、作用域与安全
└── references/                 # 10 篇按需加载
    ├── platform-detection.md       # 平台与架构判定，构建工具探测
    ├── binary-resolution.md        # 装 chsrc 的四条路径
    ├── bootstrap.md                # 死锁场景：bootstrapper → 构建 → 验证 → 交回
    ├── bootstrap-authoring.md      # 自己写 bootstrapper 的规范
    ├── source-governance.md        # 5 个动作的完整语义与退出码
    ├── scope-policy.md             # 作用域选择与 -dry 守卫的覆盖边界
    ├── ecosystem-detection.md      # 用户口语到 dish 的映射（71 个 dish）
    ├── recovery.md                 # 换源失败、回滚、解释器混淆
    ├── collaboration-workflow.md   # 何时向上游提 issue
    └── ci-validation.md            # 跨平台验证方法（记录，不执行）
```

## 🚀 使用

把 `skill/chsrc/` 放进你的 skills 目录，或指向本仓库该路径。

触发场景是「用户要求换源」，例如：

- 「帮我把 pnpm 换成国内镜像」
- 「pip 换清华源，然后告诉我怎么改回去」
- 「我这台机器 apt 现在指向哪个源」
- 「termux 怎么换源」
- 「Docker Desktop 怎么换镜像源」

不触发的场景（Skill 会明确拒绝并转做别的事）：GitHub SSH 配置、代理设置、编译项目、装某个具体包。

## 🎯 为什么需要它

这些坑都是实测出来的：

| 坑 | 后果 |
| :-- | :-- |
| `chsrc get pip -no-color` | 选项写在菜品名之后 → 报「暂不支持的换源菜品」，退出码 2 |
| 以为 `-dry` 绝对安全 | apt 旧格式路径曾绕过 dry 守卫（已提 [上游 #396](https://github.com/RubyMetric/chsrc/issues/396)）；`-dry` 覆盖绝大多数路径，但不是全部 |
| Windows 上设 `HOME` 做隔离 | 无效。chsrc 家目录取 `USERPROFILE`（`lib/xy.h:1276`） |
| 手动改 `~/.cargo/config.toml` | 绕过 chsrc 备份机制，还可能覆盖它不管理的 `[net]` 段 |
| `chsrc` 旧版不会自动落盘 | 正确处置是先升级 chsrc，手动写只作兜底 |

## ✅ 验证状态

35 个真实场景 eval，135 条断言，两组对照：

| 配置 | 通过率 |
| :-- | --: |
| 带本 Skill | **135/135（100%）** |
| 不带 Skill | 108/134（80.6%） |

覆盖 trigger 正负例、六个平台检测、bootstrap 死锁、作用域安全、容器边界、回滚。

> [!IMPORTANT]
> 验证在开发期完成，本仓库只保留 Skill 本体，不含评测脚手架。完整的验证方法与数据见 [`docs/design/chsrc-infrastructure-skill.md`](docs/design/chsrc-infrastructure-skill.md)（548 行规格）。

## 🔗 与上游的关系

本仓库是 chsrc 的**下游 Skill 包**，不包含 chsrc 源码。

发现上游问题时，本 Skill 指向对应的 issue 或 PR。上游地址：<https://github.com/RubyMetric/chsrc>（GPL-3.0-or-later）。

## 📄 许可

本 Skill 的文字内容随 chsrc 的 GPL-3.0-or-later 授权。不含 chsrc 源码。