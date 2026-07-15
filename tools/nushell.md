# Nushell 编程风格指南

本文档概述 Nushell（`nu`）脚本的编码标准和风格偏好，重点关注惯用数据管道和系统安全。

## 1. 管道优先哲学

Nushell 的核心优势是其结构化管道。**优先通过流转换数据**而非创建中间变量（`let`）。

### 不好（类 Python）
```nushell
def "fetch-sri" [url: string] {
    let data = (curl -sL $url)  # 变量开销
    let hex = ($data | hash sha256 | get sha256)
    let bytes = ($hex | decode hex)
    let b64 = ($bytes | encode base64)
    $"sha256-($b64)="
}
```

### 好（惯用）
```nushell
def "fetch-sri" [url: string] {
    ^curl -sL $url
    | hash sha256
    | decode hex
    | encode base64
    | $"sha256-($in)"
}
```

**规则**：如果逻辑是转换序列 `A -> B -> C`，写成单个管道。只在分支逻辑（`if`）或存储不同的命名结果时使用变量。

## 2. 二进制安全（`^` 转义）

从外部命令管道传输二进制数据时，始终使用**脱字符（`^`）**转义字符。

*   **为何？** 没有 `^`，Nushell 尝试将外部命令输出（stdout）解析为其内部数据结构（表、字符串）。这会损坏二进制流或对非文本数据导致"Invalid UTF-8"错误。
*   **何时？** 始终与下载工具（`curl`、`wget`）或二进制生成器（`hash`、`openssl`）一起使用，除非你明确需要 Nushell 将输出解析为表/记录。

### 示例
```nushell
# 安全：强制原始二进制流直通
^curl -sL https://example.com/file.iso | hash sha256
```

## 3. 常见模式

### 哈希
加密哈希的标准流程：
```nushell
^cat file | hash sha256 | get sha256
```

### Base64
注意 `encode base64` 自动处理填充（`=`）。不要手动追加。

### JSON/YAML/TOML 处理
如果命令未转义或输出被捕获，Nushell 自动解析 JSON/YAML/TOML。
```nushell
# 解析本地文件（自动解析）
open config.yaml | get services
```

### 配置 → 环境（`export-env` 模式）

**直接 `open toml | load-env`**，无需 `items → for → upsert` 绕圈。
```nushell
export-env {
    open 'config.toml' | load-env
}
```

**`$env.X?` 不生效** — 当前 Nushell 版本 `$env.CNTRCTL?` 仍会报 column not found。必须用 `$env | get CNTRCTL?`：
```nushell
export-env {
    const CONF = path self x.toml
    open $CONF
    | upsert CNTRCTL ($env | get CNTRCTL? | default 'docker')
    | load-env
}
```

**默认值处理合并进数据流** — 不要用单独的 `if` 分支补默认值，用 `upsert` 合并进 pipeline：
```nushell
# Bad: 分支打断数据流
open 'config.toml' | load-env
if 'CNTRCTL' not-in $env { load-env { CNTRCTL: 'docker' } }

# Good: 一次 pipeline 完成
open 'config.toml'
| upsert CNTRCTL ($env | get CNTRCTL? | default 'docker')
| load-env
```

**`path self`** — 在 `export-env` 中构造路径时用它，避免 cwd 漂移导致文件找不到。
