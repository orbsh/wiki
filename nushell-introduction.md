# Nushell：以数据为中心的现代 Shell

**结论**：CLI 复兴的最大成就。以结构化数据替代文本流，实现了 shell 层面的 SQL 语义。
**状态**：bash 的降维打击，CLI 工具链的核心枢纽。

## 一、起源：PowerShell 的灵感与超越

Nushell（2019，Jonathan Turner）明确受 PowerShell 启发。PowerShell 在 2006 年就提出「一切皆对象」——命令输出不是文本，而是 .NET 对象，管道传递的是对象引用而非字节流。这个理念超前了十年，但受限于 .NET 生态和 Windows 基因，未能在 Unix 世界普及。

Nushell 做了三件事把 PowerShell 的理念落地到 Unix 世界：

1. **用 Rust 重写**，单二进制分发，跨平台，无运行时依赖
2. **保留 Unix 互操作性**，外部命令输出仍是文本流，但 Nu 内部命令输出是结构化数据
3. **表格作为一等公民**，`ls`、`ps`、`env` 等内置命令直接输出表格，可以用 `where`、`sort-by`、`select` 等管道命令操作

## 二、核心创新：结构化数据管道

### 一切皆表格

传统 shell（bash/zsh/fish）中，命令输出是**无结构的文本流**。你用 `awk`、`sed`、`grep`、`cut` 做文本处理——本质上是在用正则表达式从非结构化文本中**猜测**结构。

Nushell 反转了这个假设：**命令输出天然是结构化的**。

```bash
# bash: 从 ps 输出中提取 PID 和 COMMAND
ps aux | awk '{print $2, $11}'
# 依赖列位置，awk 的 $2 是第二个空白分隔字段——脆弱的隐式约定
```

```nushell
# nushell: 同样的操作
ps | select pid command
# 直接按列名访问，不依赖位置，不依赖分隔符
```

区别不只是语法糖。bash 的 `awk '{print $2}'` 隐含了一个假设：输出是空格分隔的，第二列是你要的。如果 `ps` 的输出格式变了（不同版本、不同 locale），这个命令就静默失败。Nushell 的 `select pid command` 按列名访问，与输出格式无关。

### 数据类型而非字符串

Nushell 的管道传递的不是字符串，而是有类型的值：

| 类型 | 说明 | bash 等价 |
|:---|:---|:---|
| `int` | 整数 | 字符串 `'123'` |
| `float` | 浮点数 | 字符串 `'3.14'` |
| `bool` | 布尔 | 字符串 `'true'`/`'false'` 或退出码 `0`/`1` |
| `string` | 字符串 | 字符串 |
| `list` | 列表 | 空格分隔的字符串 |
| `record` | 记录（类似字典） | 无直接等价 |
| `table` | 表格（记录的列表） | 无直接等价 |
| `date` | 日期时间 | 字符串 `'2026-01-01'` |
| `filesize` | 文件大小（带单位） | 字符串 `'1.2G'` |
| `binary` | 二进制数据 | 字节流 |

bash 中一切都是字符串，类型信息丢失在文本解析中。`du -sh` 输出 `'1.2G'`，你要比较两个目录大小，得先解析单位、转换数值、再比较。Nushell 的 `ls | where size > 1gb` 直接用类型系统做比较。

### SQL 语义进 shell

Nushell 的管道命令集合本质上是一个 shell 层面的 SQL：

| Nushell 命令 | SQL 等价 | 说明 |
|:---|:---|:---|
| `select col1 col2` | `SELECT col1, col2` | 选择列 |
| `where condition` | `WHERE condition` | 过滤行 |
| `sort-by column` | `ORDER BY column` | 排序 |
| `group-by column` | `GROUP BY column` | 分组 |
| `each { ... }` | `SELECT transform(col)` | 逐行变换 |
| `reduce { ... }` | 聚合函数 | 归约 |
| `insert col expr` | `ALTER TABLE ADD COLUMN` | 插入列 |
| `update col expr` | `UPDATE ... SET col = expr` | 更新列 |
| `get col` | `SELECT col FROM ...` | 取单列 |
| `merge` | `JOIN` | 合并表 |
| `transpose` | `UNPIVOT` | 行列转置 |

这不是"借鉴 SQL 语法"，而是**同一个抽象在不同层面的实例化**。SQL 操作关系表，Nushell 操作 shell 命令输出——两者的输入输出都是结构化的行列表，操作集合自然重合。

### Polars 集成：Dataframe 进 shell

Nushell 通过 `polars` 插件将 Polars DataFrame 引入 shell 环境。这意味着你可以在命令行中做：

```nushell
# 读取 CSV，过滤，分组聚合，排序
open sales.csv
| into df
| where (get amount) > 100
| group-by category
| agg { (get amount) | sum }
| sort-by amount --reverse
| into table
```

这是 **OLAP 查询在 shell 层面的实现**。传统做法是写 Python 脚本（pandas/polars）或启动 SQL 客户端。Nushell + Polars 让你在管道中直接做数据分析，不需要启动额外的进程。

## 三、语言设计：Rust 的降维打击

Nushell 的语法设计明显受 Rust 影响，相对于 bash 是降维打击：

### 变量与作用域

```bash
# bash: 变量无作用域，全局污染
my_var="hello"
# 子 shell 中修改不影响父 shell
(subshell_var="changed")
echo $my_var  # 仍是 "hello"
```

```nushell
# nushell: 词法作用域，显式声明
let my_var = "hello"
mut counter = 0
$counter += 1
```

### 错误处理

```bash
# bash: 错误被静默忽略，`set -e` 是全局补丁
rm non_existent_file  # 无输出，无错误（除非 set -e）
```

```nushell
# nushell: 错误是值，可以被捕获和处理
rm non_existent_file | complete  # 返回 { exit_code: 1, stdout: "", stderr: "..." }
try { rm non_existent_file } catch { |e| print $"Error: ($e.msg)" }
```

### 函数定义

```bash
# bash: 函数体是字符串，缩进是陷阱
my_func() {
    echo "hello"  # tab 和空格混用是常见 bug
}
```

```nushell
# nushell: 结构化函数定义，带类型签名
def greet [name: string] {
    print $"Hello, ($name)!"
}

def "str max-len" [] -> int {
    $in | str length | math max
}
```

### 模式匹配与解构

```nushell
# nushell 支持模式匹配
match (ls | first) {
    {type: "file"} => { print "It's a file" }
    {type: "dir"} => { print "It's a directory" }
    _ => { print "Unknown" }
}
```

## 四、自动补全：函数即补全

Nushell 的自动补全系统最大的突破不是功能多，而是**自定义补全就是一个普通函数**——可选接收命令上下文，输出一个列表。没有特殊 API，没有状态机，没有底层 `compadd`/`compset`。

### 自定义补全：函数即补全

```nushell
# 自定义补全就是一个 def，返回列表
def "nu-complete my-commands" [] {
    [list get set delete]
}

# 可选接收上下文（当前已输入的参数、子命令等）
def "nu-complete git-branch" [context: string] {
    ^git branch --format='%(refname:short)'
    | lines
    | where { $in | str starts-with $context }
}

# 在命令签名中引用
def my-cmd [
    command: string@nu-complete my-commands   # 静态列表补全
    branch: string@nu-complete git-branch      # 动态上下文补全
] { }
```

补全函数和普通函数没有区别：`def`、参数、返回值。补全系统只是调用这个函数，拿到列表，展示给用户。这和 bash/zsh 的补全是两个世界。

### 对比 bash/zsh：特殊 API 的噩梦

bash/zsh 的自定义补全需要调用**专门的补全 API**，理解一整套状态机概念：

```bash
# bash: 自定义补全需要特殊函数 + compgen + compopt
_my_cmd_completions() {
    local cur="${COMP_WORDS[COMP_CWORD]}"
    COMPREPLY=( $(compgen -W "list get set delete" -- "$cur") )
}
complete -F _my_cmd_completions my_cmd
```

```zsh
# zsh: 更复杂的 _arguments spec 语法
_my_cmd() {
    _arguments \
        '1:command:(list get set delete)' \
        '2:branch:_git_branch'
}
compdef _my_cmd my_cmd
```

bash/zsh 的问题：
- 需要学习 `compgen`、`compopt`、`COMPREPLY`、`compdef`、`_arguments` 等专用 API
- 补全函数和普通函数是**两个体系**，不能复用
- 上下文传递（当前输入了几个参数、在哪个子命令）需要手动解析 `COMP_WORDS`
- zsh 的 `_arguments` spec 语法本身就是一门小语言

Nushell 把这些全部消除了：补全就是函数，输入是上下文（可选），输出是列表。没有额外概念。

### 多维补全

- **命令补全**：内置命令 + 外部命令 + 子命令
- **参数补全**：根据命令签名自动补全参数名
- **值补全**：补全文件路径、列名（从管道上下文推断）、Git 分支、环境变量
- **动态补全**：补全候选由函数动态生成（如上例 `nu-complete git-branch`）

### 终端集成

Nushell 的自动补全通过 `carapace` 或内置补全引擎工作，支持：
- 基于上下文的补全（知道你在哪个子命令的哪个参数位置）
- 补全菜单的多列展示
- 补全描述（候选旁边显示帮助文本）

## 五、Unix 互操作性：保留基础，突破上限

Nushell 没有放弃 Unix 的文本互操作性，而是在其上构建了结构化层：

### 向下兼容

```nushell
# 外部命令输出仍是文本，Nu 自动尝试解析
git log --oneline  # 自动解析为表格（hash, message 列）

# 如果解析失败，回退为字符串
some-unknown-command | get 0  # 返回原始文本

# 强制保留二进制流
^curl -sL $url | save response.bin  # ^ 转义，跳过 Nu 的解析
```

### 向上突破

```
┌─────────────────────────────────────────────────────┐
│                    Unix 互操作层                      │
│  一切皆文本 → 基础互操作性保证                         │
├─────────────────────────────────────────────────────┤
│                  Nushell 结构化层                     │
│  一切皆表格 → 高级互操作性突破                         │
├─────────────────────────────────────────────────────┤
│                  Polars 集成层                        │
│  DataFrame → shell 层面的 OLAP                       │
└─────────────────────────────────────────────────────┘
```

**Unix 的「一切皆文本」保证了基础的互操作性**——任何两个程序可以通过文本流通信，这是 Unix 哲学的伟大之处。但这也**阻碍了更高级的互操作**——你无法在管道中传递类型信息、结构化数据、错误上下文。Nushell 的突破在于：保留文本互操作的底线，同时在上层提供结构化的互操作能力。

## 六、整合趋势：替代而非共存

Nushell 的趋势不是"和现有工具共存"，而是**整合替代**。更高的集成度 = 更低的运维复杂度 = 更小的摩擦。

### 内置格式处理：消除 jq/yq

Nushell 内置 JSON、YAML、TOML、CSV、XML 等格式的解析和生成。`open config.json | get services.web.port` 直接操作，不需要 `jq '.services.web.port'`。`open data.csv | where status == "active"` 直接过滤，不需要 `yq`。

很多工具还在被使用，是因为**惯性**：AI 默认用 `jq` 处理 JSON，k8s 生态默认用 `yq` 处理 YAML。但 Nushell 已经原生覆盖了这些场景——再安装 `jq`/`yq` 就像在 Nushell 环境里装 `grep` 一样多余。

| 格式 | 传统工具 | Nushell 等价 |
|:---|:---|:---|
| JSON | `jq '.key'` | `open file.json \| get key` |
| YAML | `yq '.spec.image'` | `open pod.yaml \| get spec.image` |
| TOML | `tomlq '.server.port'` | `open config.toml \| get server.port` |
| CSV | `csvtool col 1 file.csv` | `open data.csv \| get column1` |

### Polars 插件：数据科学环境的整合

Polars 作为 Nushell 插件的意义不止是"多了一个 DataFrame 工具"——它是**将数据科学环境整合进 shell** 的关键一步。

**对比 Python Polars**：Python 的 Fluent API（管道化链式调用）在语法层面受限——不能随意换行，每行末尾需要转义符 `\` 或用括号包裹。Nushell 的管道语法天然支持链式调用，每一步独占一行，无转义、无括号：

```python
# Python Polars: 需要括号包裹或行尾转义
result = (
    pl.scan_csv("sales.csv")
    .filter(pl.col("amount") > 100)
    .group_by("category")
    .agg(pl.col("amount").sum())
    .sort("amount", descending=True)
    .collect()
)
```

```nushell
# Nushell + Polars: 天然管道，每步一行
open sales.csv
| into df
| where (get amount) > 100
| group-by category
| agg { (get amount) | sum }
| sort-by amount --reverse
```

**对比 SQL**：SQL 是声明式的，但缺乏 shell 集成——你需要启动数据库客户端、导入数据、写查询、导出结果。Nushell + Polars 直接在管道中操作，零启动开销，与文件系统、网络请求、其他 CLI 工具无缝衔接。

Polars 插件如果足够完善，Nushell 可以成为一个**比 Python 好用、比 SQL 好用的数据科学环境**，直接与 DuckDB 竞争。DuckDB 的优势是嵌入式 OLAP，Nushell + Polars 的优势是**同时是 shell 和数据分析环境**——不需要在"写脚本"和"做分析"之间切换。

### 整合的逻辑

Nushell 本身是一个**多层松散体系**：核心（语言引擎 + 数据类型系统）→ 内置工具（`open`、`get`、`where` 等原生命令）→ 插件（Polars、外部命令集成）→ 脚本（用户自定义 `def`）。核心和内置工具打包到单个可执行文件，这是物理形式的选择——方便部署和分发（一个二进制、零依赖），也有 Rust 生态的影响（静态链接、无运行时）。

这种整合是**用统一的数据模型消除工具间的格式转换成本**。当你有了结构化数据管道，jq/yq/csvtool 这些"格式转换中间人"就失去了存在的理由。Polars 插件同理——不是为了"内置 Polars"，而是为了让 DataFrame 操作融入 shell 管道，消除"shell → Python → shell"的上下文切换。

## 结论

Nushell 是 CLI 复兴的最大成就，因为它解决了一个根本问题：**shell 的数据模型错了**。bash 把一切视为文本，这在 1979 年（Unix V7）是正确的选择（当时没有更好的抽象），但在 2026 年是过时的限制。Nushell 用结构化数据替代文本流，用 Rust 语法替代 POSIX shell 语法，用 SQL 语义替代 awk/sed/grep 的文本处理——这不是渐进式改进，是范式替换。

PowerShell 在 2006 年就看到了这一点，但受限于 .NET 生态未能普及。Nushell 用 Rust 重写，跨平台，无依赖，Unix 原生——终于把这个理念带到了它应该在的地方。

---

## 交叉引用

- **[可视化迷思](visual-myth.md)**：Nushell 是文本形式在 AI 时代结构性优势的实证——LLM 输出文本，Nushell 消费结构化文本，中间零格式损耗。
- **[现代语言设计](modern-language-design.md)**：Nushell 的语法设计（显式 let、词法作用域、错误处理）是现代语言设计趋势在 shell 层面的实例化。
- **[Nushell 编码风格](tools/nushell.md)**：Nushell 脚本的编码标准和惯用模式。
