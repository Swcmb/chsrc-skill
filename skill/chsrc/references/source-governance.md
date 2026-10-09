# 五个动作的完整语义

`get` / `set` / `reset` / `measure` / `list` 是 chsrc 的全部换源动作，另有 `help`、`version`、
`issue`。本文给出每个动作的输入、输出、是否写盘、退出码。

**铁律：换完源必须跑 `chsrc get <dish>` 复查。** 详见文末「复查」。

---

## 一、完整别名表

别名全部在 `src/chsrc-main.c` 的分派链上做字符串全等比较，写全名最安全。

| 动作 | 全部别名 | 分派处 |
| :-- | :-- | :-- |
| 查看当前源 | `get` · `g` | `src/chsrc-main.c:371-372` |
| 换源 | `set` · `s` | `src/chsrc-main.c:395-396` |
| 恢复默认源 | `reset` · `rest` · `r` | `src/chsrc-main.c:426-428` |
| 测速 | `measure` · `mea` · `m` · `cesu` · `ce` · `c` | `src/chsrc-main.c:340-345` |
| 列出信息 | `list` · `l` · `ls` | `src/chsrc-main.c:274-276` |
| 帮助 | `help` · `h` · `-h` · `-help` · `--help` | `src/chsrc-main.c:252-256` |
| 版本 | `version` · `ver` · `-v` · `-version` · `--version` | `src/chsrc-main.c:263-267` |
| 相关 issue | `issue` · `issues` · `isue` · `i` | `src/chsrc-main.c:453-456` |

注意两处与 `SKILL.md` 的差异，以本文为准（源码为准）：

- `list` 还有别名 `l`，`version` 还有别名 `ver`。
- `c` 与 `m` 都指向 `measure`。`r` 指向 `reset`。

无参数运行时打印帮助并返回 0（`src/chsrc-main.c:119-123`）。

---

## 二、位置参数规则：选项必须写在 dish 名之前

解析器从 `argv[1]` 遍历到 `argv[argc]`，每遇到一个以 `-` 开头的参数，就把
`cli_arg_Dish_pos` 与 `cli_arg_Mirror_pos` 各加一（`src/chsrc-main.c:148-240`，
自增在 `:237-238`）。两个位置的初值是 2 和 3（`src/chsrc-main.c:129-130`）。

因此选项写在 dish 名后面会让 dish 位置整体后移，选项自己被当成菜品名：

```bash
chsrc get -no-color pip     # 正确：argv[1]=get, argv[2]=-no-color, argv[3]=pip
chsrc get pip -no-color     # 错误：argv[2]=pip 被当作选项跳过，argv[3]=-no-color 被当作菜品名
```

第二种写法报「暂不支持的换源菜品」并返回 2（`src/chsrc-main.c:470-477`）。

`-scope` 有两种写法，空格写法额外占用一个参数位，所以它让 `cli_arg_Dish_pos` 与
`cli_arg_Mirror_pos` 各加二；等号写法只加一（`src/chsrc-main.c:166-175`）。两种都合法：

```bash
chsrc set -scope=user pip tuna      # 空格写法
chsrc set -scope=user pip tuna      # 等号写法效果相同
```

### 全部合法选项

| 选项 | 生效行为 | 位置 |
| :-- | :-- | :-- |
| `-ipv6` | 测速走 IPv6 | `src/chsrc-main.c:152-155` |
| `-local` | **已弃用**，置 `ProjectScope` 并打 deprecation warning | `src/chsrc-main.c:156-162` |
| `-scope` / `-scope=` | 作用域，值取 `project`/`user`/`system`/`default`，大小写不敏感 | `src/chsrc-main.c:163-206` |
| `-en` / `-english` | 英文输出 | `src/chsrc-main.c:207-210` |
| `-dry` | Dry Run | `src/chsrc-main.c:211-214` |
| `-no-color` / `-no-colour` | 关闭颜色 | `src/chsrc-main.c:215-219` |
| `-h` / `-help` / `--help` | 转成 `help` 命令 | `src/chsrc-main.c:220-225` |
| `-v` / `-version` / `--version` | 转成 `version` 命令 | `src/chsrc-main.c:226-231` |

作用域值非法时报「无效的换源作用域」并返回 2（`src/chsrc-main.c:192-205`）。
其它任何以 `-` 开头的参数报「未识别的命令行选项」并返回 2（`src/chsrc-main.c:232-236`）。

---

## 三、五个动作逐一

### get — 查看当前源

| 项 | 值 |
| :-- | :-- |
| 输入 | dish 名，无第二参数 |
| 输出 | 当前源状态 |
| 写盘 | 只读 |
| 退出码 | 正常 0（`src/chsrc-main.c:391`）；缺 dish 名 2；未知 dish 2 |

dish 未实现 `getfn` 时打印「暂未对 X 实现 get 功能，邀您帮助: chsrc issue」，**不退出 2**，
继续走完流程后仍返回 0（`src/framework/waiter.c:788-795`）。因此**退出码 0 不代表
`get` 读到了源**，判断依据要看输出文本。

执行体是 `waiter_handle_Get_Source()`（`src/framework/waiter.c:773`），最终调用
`dish->getfn("")`。完成后追加一段简短的维护信息（食谱更新日期、后厨人数）（`src/framework/waiter.c:963-981`）。

### set — 换源

| 项 | 值 |
| :-- | :-- |
| 输入 | dish 名 + 可选的镜像站 code 或完整 URL |
| 输出 | 选中镜像站、写入的文件名、换源类型小结 |
| 写盘 | **写** |
| 退出码 | 正常 0（`src/chsrc-main.c:422`）；缺 dish 名 2；未知 dish 2 |

选出源的逻辑集中在 `chsrc_yield_source()`（`src/framework/source.c:526-583`），
五种入口：

1. 给了 URL 且 dish 支持 `can_user_define` → 用该 URL（`src/framework/source.c:555-565`）
2. 给了 URL 但 dish 不支持自定义源 → 退出 3（`src/framework/source.c:557-562`）
3. 给了镜像站 code → 从该 dish 的源数组里找 code 匹配的项（`src/framework/source.c:567-577`）
4. code 是 `first` → 直接取维护团队测速排第一的源，即 `sources[1]`（`src/framework/dish.c:253-258`）
5. 什么都没给 → 测速后取最快的（`src/framework/source.c:578-581`）

镜像站 code 存在但该 dish 没有这个源的源时退出 4，code 不存在时同样退出 4，两种情况
都会提示用 `chsrc list <dish>` 查看可用源（`src/framework/dish.c:266-307`）。

第二个位置参数的判定只有两条：`hp_is_url()` 判断前缀是 `http://` 或 `https://`（`src/framework/helper.c:15-20`），
是则当 URL 用，否则当镜像站 code 用（`src/framework/waiter.c:808-815`）。

### reset — 恢复默认源

| 项 | 值 |
| :-- | :-- |
| 输入 | dish 名，无第二参数 |
| 输出 | 「已重置为上游默认源」 |
| 写盘 | **写** |
| 退出码 | 正常 0（`src/chsrc-main.c:449`）；缺 dish 名 2；未知 dish 2 |

`reset` 会置 `ProgMode.ResetMode = true`（`src/chsrc-main.c:438`）。该标志让选源逻辑
直接返回 `sources[0]`（上游默认源固定在数组第一位），跳过测速（`src/framework/source.c:352-353`）。

dish 未实现 `resetfn` 时打印「暂未对 X 实现 reset 功能」并返回 0（`src/framework/waiter.c:901-908`）。
对只有上游默认源的 dish 强行换源时，`chsrc_confirm_source()` 报「暂未对该菜品实现重置」
并退出 3（`src/framework/source.c:604-609`）。

### measure — 测速

| 项 | 值 |
| :-- | :-- |
| 输入 | dish 名 |
| 输出 | 每个 provider 的测速结果、最快镜像站名、该源的 URL |
| 写盘 | 只读 |
| 退出码 | 正常 0（`src/chsrc-main.c:366`）；缺 dish 名 2；未知 dish 2 |

置 `ProgMode.MeasureMode = true`（`src/chsrc-main.c:354`），该标志让输出额外打印一行
「镜像源地址: <url>」（`src/framework/source.c:444-448`）。

底层调 `curl`（`src/framework/source.c:98-101`）：

```text
curl -qsL [--ipv6] -o <devnull> -w "%{http_code} %{speed_download}" -m8 -A <User-Agent> <speed_measure_url>
```

`%{speed_download}` 被解析成 Byte/s 再换算成人类可读格式（`src/framework/source.c:113-134`）。
测速结果里出现「精准测速」/「模糊测速」标记，取决于该源是否带专用测速链接
（`src/framework/source.c:284-295`）。

### list — 列出信息

| 项 | 值 |
| :-- | :-- |
| 输入 | 可选 dish 名 |
| 输出 | 见下 |
| 写盘 | 只读 |
| 退出码 | 一律 0（`src/chsrc-main.c:333`） |

- **不带 dish 名**：打印全部可用镜像站（code / 简写 / URL / 名称）（`src/chsrc-main.c:278-283`，
  `src/framework/waiter.c:16-40`）＋ 全部支持换源的菜品名。
- **`list mirrors` / `list mirror`**：只打印镜像站表（`src/chsrc-main.c:287-292`）。
- **`list dishes` / `list dish` / `list dishs`**：只打印菜品表（`src/chsrc-main.c:294-300`）。
- **`list os` / `list lang`（`pl`/`language`）/ `list ware`（`software`）**：按菜单分类打印（`src/chsrc-main.c:302-321`）。
- **`list <dish>`**：该 dish 的可用源表（code / 换源 URL / 镜像站名，`src/framework/waiter.c:700-719`）、
  可用功能（Get / Reset / UserDefine / 三种作用域的支持度，`src/framework/waiter.c:193-272`）、
  维护信息（`src/framework/waiter.c:723-727`）。

`list` 缺 dish 名不算错误，也返回 0。别拿 `list` 的退出码判断菜品名是否存在。

---

## 四、`-dry` 是选项，不是命令

**没有 `dry` 命令**，也没有 `verify` 命令。`-dry` 写在选项位：

```bash
chsrc set -dry -scope=user pip     # 预览
chsrc set -scope=user pip          # 实际执行
chsrc get pip                      # 复查
```

`-dry` 生效时有三层拦截：选源时跳过测速直接取 `sources[1]`（`src/framework/source.c:370-374`）；
执行外部命令时在 `system()` 之前直接返回（`src/framework/OS.c:324-327`）；
写文件时跳到日志分支（`src/framework/OS.c:750`、`:805`、`:836`、`:875`）。
因此 `-dry` 下不会改动任何磁盘状态，副作用是外部命令的失败退出码 6 也不会触发
（`src/framework/OS.c:342-347`）。启动时会打一行 Dry Run 提示（`src/chsrc-main.c:242-249`）。

`verify` 语义由 `chsrc get <dish>` 承担。

---

## 五、退出码

定义在 `src/framework/base.c:162-168`。

| 码 | 宏 | 含义 |
| --: | :-- | :-- |
| 0 | `Exit_OK` | 正常结束 |
| 1 | `Exit_Fatal` | 致命错误 |
| 2 | `Exit_Unknown` | 未知菜品或参数 |
| 3 | `Exit_Unsupported` | 该 dish 不支持此操作 |
| 4 | `Exit_UserCause` | 用户侧问题，多为缺依赖 |
| 5 | `Exit_MaintainerCause` | recipe 自身缺陷 |
| 6 | `Exit_ExternalError` | 外部命令执行失败 |

各动作实际会吐出的码：

| 场景 | 码 | 出处 |
| :-- | --: | :-- |
| `measure`/`get`/`set`/`reset` 缺 dish 名 | 2 | `src/chsrc-main.c:347-353`、`:374-380`、`:398-404`、`:430-436` |
| 未知 dish | 2 | `src/chsrc-main.c:470-477` |
| 未知命令 | 2 | `src/chsrc-main.c:462-468` |
| 未知选项 | 2 | `src/chsrc-main.c:232-236` |
| 作用域值非法 | 2 | `src/chsrc-main.c:192-205` |
| dish 不支持用户自定义 URL | 3 | `src/framework/source.c:557-562` |
| 套餐不支持自定义 URL / 指定镜像站 | 3 | `src/framework/waiter.c:823-828`、`:830-838` |
| 对该菜品未实现 reset | 3 | `src/framework/source.c:604-609` |
| 缺 curl | 4 | `src/framework/source.c:380-387` |
| Cygwin 版 curl | 4 | `src/framework/source.c:389-406` |
| 指定作用域不被 dish 支持 | 4 | `src/framework/source.c:797-840` |
| 镜像站 code 不存在 / 该 dish 无此源 | 4 | `src/framework/dish.c:289-306` |
| 套餐所有子菜品都无指定镜像站的源 | 4 | `src/framework/source.c:508-510` |
| dish 源列表为空 | 5 | `src/framework/source.c:362-368`、`src/framework/dish.c:219-233` |
| 源 URL 为空 | 5 | `src/framework/source.c:610-616` |
| 外部命令返回非零 | 6 | `src/framework/OS.c:342-347` |

退出码由第一个触发的 `exit()` 决定。`0` 只代表流程走完，`get`/`reset` 遇到未实现的
dish 也会返回 0（见上文两处）。

---

## 六、curl 是测速的硬依赖

测速前检查外部命令 `curl`，缺失即报错「没有curl命令，无法测速」并退出 4
（`src/framework/source.c:380-387`）。Windows 上还会跑一次 `curl --version`，命中
`pc-cygwin` 就报「你使用的是Cygwin构建的curl，该版本的curl存在bug」并退出 4
（`src/framework/source.c:389-406`）。

`set` 不带第二参数时也走测速，因此同样需要 curl。给全名镜像站 code 可以完全绕过测速，
这也是指定 `tuna` 之类固定站点比让 chsrc 自己挑更可控的原因。

`-dry` 在 curl 检查之前就返回了，所以 `-dry` 预览在无 curl 的机器上也能跑。

---

## 七、combo dish 的递归行为

套餐 dish（`py_group` / `js_group` / `uv`）本身无 `getfn`/`setfn`/`resetfn`，
`waiter_handle_user_command()` 在开头先打印一份子菜品清单：
「套餐 X 由 N 个子菜品组成: a, b, c」（`src/framework/waiter.c:922-940`）。
之后各动作的差异很大：

| 动作 | 对 combo dish 的行为 | 出处 |
| :-- | :-- | :-- |
| `get` | 逐个调用子菜品的 `getfn`，输出按子菜品分组 | `src/framework/waiter.c:775-786` |
| `set` | `push_combo_stack` 后逐个递归 set，`option` 原样传给每个子菜品 | `src/framework/waiter.c:817-851` |
| `reset` | 逐个递归 reset，**不** push combo 栈 | `src/framework/waiter.c:888-899` |
| `measure` | **只列出子菜品名，不测速**，提示逐个手动测速 | `src/framework/waiter.c:737-760` |
| `list` | 逐个列出子菜品的维护信息，嵌套 combo 继续递归 | `src/framework/waiter.c:634-689` |

`set` 套餐时有两道前置拦截（`src/framework/waiter.c:821-838`）：

- 给了自定义 URL 且套餐的 `all_sub_dishes_use_same_source` 为假 → 退出 3，提示逐个子菜品单独指定。
- 给了镜像站 code 且没有任何子菜品拥有该站的源 → 退出 3。

逐个递归时，某个子菜品没有所请求镜像站的源会打印提示「当前子菜品没有来自所请求镜像站的源，
尝试测速选择最快源」并退化为测速（`src/framework/source.c:500-505`）。套餐全部子菜品都没有时退出 4
（`src/framework/source.c:508-510`）。

`all_sub_dishes_use_same_source` 为真时，套餐第一个子菜品测出的源会被缓存复用给整个套餐
（`src/framework/source.c:451-458`）。

---

## 八、复查

`get` 是只读的，任何时候都能跑。三个典型失败场景都不会让 `set` 报错退出：

- 测速选中了另一个源——输出里「最快镜像站:」一行才是实际写入的站。
- 写入被权限拒绝——系统层报错，非 chsrc 的退出码路径。
- 配置被别的工具覆盖——chsrc 无从感知。

所以换源流程固定为三步：

```bash
chsrc set -dry -scope=user pip tuna    # 预览写入路径与文件名
chsrc set -scope=user pip tuna          # 执行
chsrc get pip                           # 读回当前源，与预期比对
```

不跑第三步等于没验证。前两步返回 0、`get` 的输出与预期不一致时，先看
`references/recovery.md`。

---

## 九、错误消息速查

grep 源码时按这些字符串定位：

| 消息 | 出处 |
| :-- | :-- |
| 暂不支持的换源菜品。 | `src/chsrc-main.c:474` |
| 不支持的命令 | `src/chsrc-main.c:464` |
| 未识别的命令行选项 | `src/chsrc-main.c:234` |
| 无效的换源作用域 | `src/chsrc-main.c:201` |
| 请提供想要查看/设置/重置/测速的菜品名。 | `src/chsrc-main.c:376`、`:400`、`:432`、`:349` |
| 暂未对 X 实现 get/set/reset 功能，邀您帮助 | `src/framework/waiter.c:794`、`:877`、`:907` |
| 没有curl命令，无法测速 | `src/framework/source.c:383` |
| 选中镜像站: | `src/framework/source.c:619` |
| 最快镜像站: | `src/framework/source.c:437` |
| 已重置为上游默认源 | `src/framework/source.c:680` |
| 全自动换源完成 / 半自动换源完成 | `src/framework/source.c:696`、`:716` |
| 因实现约束需按上述提示手工操作 | `src/framework/source.c:658-659` |

「全自动 / 半自动 / 手工」三档由 recipe 自己通过 `chsrc_determine_chgtype()` 上报
（`src/framework/source.c:637-640`）。半自动与手工意味着 chsrc 只改了部分内容，
需要按提示补完——这两种输出出现时，`get` 的结果往往只反映已完成的部分。