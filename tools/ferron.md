# Ferron 代码风格与重构指南

## 风格

### 管道优先

```nu
# ✓ 管道尾绑定
$env.PATH_INFO? | default '' | str trim -c '/' | let pi
# ✗ 传统 let
let pi = ($env.PATH_INFO? | default '' | str trim -c '/')
```

### 字符串裁剪

```nu
# ✓ 安全：首尾都处理
str trim -c '/'
# ✗ 脆弱：只处理开头，越界会错
str substring 1..
```

### 变量复用

计算一次，复用多次：

```nu
$env.PATH_INFO? | default '' | str trim -c '/' | let pi
# pi 用于推导前缀，也用于最终路径拼接
```

## 重构

### 抽象挂载点

脚本不应知道自己的挂载路径。从环境变量自动推导：

```
挂载前缀 = REQUEST_URI - PATH_INFO
```

### 合并函数

被单一调用者使用的辅助函数，直接合并进调用者，减少间接层。
