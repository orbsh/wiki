# Debian 容器内沙箱方案：bwrap vs Landlock

## 架构背景

在 Debian Docker 容器内部运行 hermes-agent 并实现多租户隔离，属于"容器内嵌沙箱（Container-in-Container）"架构。

Debian 12 (Bookworm) 起采用 **Merged-/usr** 架构：`/lib`, `/lib64`, `/bin`, `/sbin` 本质上都是指向 `/usr` 下对应目录的软链接。这对 bwrap 方案有直接影响（见下文）。

---

## 方案一：Bubblewrap (bwrap)

### 原理

外部命令拉起沙箱，通过 `--ro-bind` / `--bind` 等参数精确控制挂载点，构建隔离的文件系统视图。

### Debian Merged-/usr 陷阱

在构建 bwrap 参数时，**必须显式在沙箱内部创建对应的软链接**（`--symlink`），否则动态链接器报错找不到 `python3` 或 `sh`。

### Python 封装代码（Debian 容器优化版）

```python
import asyncio
import os
from typing import List, Dict

class DebianContainerSandbox:
    def __init__(self, tenant_id: str, base_data_dir: str = "/data/tenants"):
        self.tenant_id = tenant_id
        self.tenant_dir = os.path.abspath(os.path.join(base_data_dir, tenant_id))
        os.makedirs(self.tenant_dir, exist_ok=True)

    def _build_bwrap_args(self, cmd_to_run: List[str], env_vars: Dict[str, str]) -> List[str]:
        args = ["bwrap"]

        # 1. 基础命名空间隔离
        # 注意：如果 hermes 需要请求云端大模型（OpenAI等），不要加 --unshare-net
        args += ["--unshare-pid", "--unshare-ipc", "--unshare-uts", "--unshare-cgroup"]

        # 2. 【核心：适配 Debian Merged-usr 架构】
        args += ["--ro-bind", "/usr", "/usr"]
        args += ["--ro-bind", "/etc/ssl", "/etc/ssl"]  # 必须，否则 HTTPS 请求失败
        # 关键补丁：在沙箱 tmpfs 根目录中手动创建 Debian 缺失的软链接
        args += ["--symlink", "usr/bin", "/bin"]
        args += ["--symlink", "usr/sbin", "/sbin"]
        args += ["--symlink", "usr/lib", "/lib"]
        if os.path.exists("/lib64"):
            args += ["--symlink", "usr/lib64", "/lib64"]

        # 3. 挂载 Python 第三方库路径 (site-packages / dist-packages)
        if os.path.exists("/usr/local"):
            args += ["--ro-bind", "/usr/local", "/usr/local"]

        # 4. 核心内核虚拟文件系统
        args += ["--proc", "/proc", "--dev", "/dev"]

        # 5. 阅后即焚临时目录
        args += ["--tmpfs", "/tmp"]

        # 6. 租户数据隔离：映射专属目录并指定为 HERMES_HOME
        args += ["--bind", self.tenant_dir, "/home/hermes"]
        env_vars["HERMES_HOME"] = "/home/hermes"

        for k, v in env_vars.items():
            args += ["--setenv", k, v]

        args += ["--die-with-parent"]
        args += cmd_to_run
        return args

    async def execute(self, prompt: str):
        cmd = ["python3", "-m", "hermes_agent", "chat", "--message", prompt]
        env = {
            "PATH": os.environ.get("PATH", "/usr/local/bin:/usr/bin:/bin"),
            "LANG": "C.UTF-8"
        }
        final_args = self._build_bwrap_args(cmd, env)
        process = await asyncio.create_subprocess_exec(
            *final_args,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )
        stdout, stderr = await process.communicate()
        if process.returncode == 0:
            return stdout.decode().strip()
        else:
            raise RuntimeError(f"Debian 沙箱运行失败: {stderr.decode()}")
```

### 排坑指南

#### 1. 外层 Docker 必须赋予 SYS_ADMIN 权限

bwrap 依赖 Linux 内核命名空间创建权限，默认 Docker 容器禁用了这些系统调用。

```bash
docker run -d \
  --name my-hermes-service \
  --cap-add=SYS_ADMIN \
  -v /宿主机数据盘:/data/tenants \
  debian:bookworm
```

（安全不敏感场景可用 `--privileged`，但推荐最小权限 `--cap-add=SYS_ADMIN`）

#### 2. Debian dist-packages 目录

Debian 系统 Python 特有魔改：通过 `apt install python3-pip` 安装的全局库放在 `/usr/local/lib/python3.x/dist-packages/`（不是常规 `site-packages`）。

- 上述代码已通过 `--ro-bind /usr/local /usr/local` 完整映射
- 若使用 venv/Poetry，需额外 `--ro-bind /opt/venv /opt/venv` 并修改 `PATH` 指向虚拟环境 `bin` 目录

---

## 方案二：Landlock（内核级自限制沙箱）

### 原理

Landlock 是 Linux 5.13+ 引入的 LSM（Linux Security Module）。与 bwrap 从外部构建监狱不同，Landlock 让进程**从内部自我剥夺权限**——Python 代码自己锁死自己的文件系统访问范围。

### 核心工作流

1. **定义规则**：声明"我只信任 `/data/tenants/tenant_001` 这个专属目录"
2. **强制实施（Enforce）**：调用 Landlock 系统调用
3. **闭环运行**：一旦激活，该进程及所有子进程被内核剥夺访问未授权目录的权限（即使 root 也无法越界）

### 优势（AI Agent / 多租户场景）

| 维度 | bwrap | Landlock |
|------|-------|----------|
| 外部依赖 | 需安装 bubblewrap，小心挂载系统库 | 零依赖，纯内核系统调用 |
| 特权需求 | 需 `SYS_ADMIN` | **Rootless**，普通用户即可调用 |
| Debian Merged-/usr | 必须手动 `--symlink` 补丁 | 无需处理（Python 启动时已加载库） |
| 隔离粒度 | 目录级挂载 | **文件级**精确控制（读/写/执行分别授权） |
| 进程/网络隔离 | ✅ 支持 | ❌ 仅文件系统 |

### Python 代码示例

```python
import os
from github_or_pip_library import landlock  # 轻量封装库，或直接用 ctypes/os 调用

def run_tenant_agent(tenant_id, prompt):
    tenant_dir = f"/data/tenants/{tenant_id}"

    # 1. 允许 Python 解释器读取标准库和核心命令（只读）
    allowed_paths = [
        (landlock.ACCESS_FS_READ_FILE, "/usr"),
        (landlock.ACCESS_FS_READ_FILE, "/lib"),
        (landlock.ACCESS_FS_READ_FILE, "/etc/ssl"),  # SSL 证书（联网大模型必需）
    ]

    # 2. 赋予该租户专属目录【读写】最高权限
    allowed_paths.append(
        (landlock.ACCESS_FS_READ_FILE | landlock.ACCESS_FS_WRITE_FILE, tenant_dir)
    )

    # 3. 吞下"自锁胶囊"——激活 Landlock 规则
    # 从这一行往后，内核全面接管风控
    landlock.restrict_self(allowed_paths)

    # 4. 安全地将控制权移交给 hermes-agent
    # 即使 Prompt Injection 试图执行：
    #   with open('/data/tenants/another_tenant/secret.txt') as f:
    # 内核瞬间拒绝，抛出 PermissionError
    print(f"租户 {tenant_id} 的安全沙箱已启动，正在执行任务...")
    # 执行具体的 agent 逻辑 ...
```

> **注**：Python 3.13 已开始原生支持部分 Landlock 接口，也可通过 `ctypes` 直接调用系统调用。

### Landlock 的路径重定向限制

**核心概念**：Landlock 是"看门人（Access Controller）"，不是"幻术师（File Virtualization）"。它**绝对不能**做路径重定向（路径别名/虚拟映射）。

#### 为什么 Landlock 不能做别名映射？

Landlock 的底层设计逻辑：路径在操作系统里长什么样，它就按原样去拦截或放行。
- 无法改变 Linux 内核的 VFS（虚拟文件系统）路径树
- 如果试图在 Landlock 锁定的沙箱里访问 `~/.hermes`，而之前没有显式把 `~/.hermes` 物理路径放进白名单，内核直接返回 `Permission Denied`——它根本不知道你想把它重定向到 `/data/tenants/tenant_002`

#### 在 Landlock 架构下实现路径映射的三种方案

**方案 A：修改环境变量（推荐，高并发安全）**

hermes-agent 支持通过 `HERMES_HOME` 环境变量改变主干目录。在调用 Landlock 之前动态修改环境变量即可：

```python
import os

tenant_dir = "/data/tenants/tenant_002"
os.environ["HERMES_HOME"] = tenant_dir  # 直接让组件不再寻找 ~/.hermes

# 随后正常初始化 Landlock 白名单
# 允许读写 tenant_dir ...
# ruleset.restrict()
```

**方案 B：符号链接（单实例/进程隔离模式）**

如果第三方库写死了只能访问 `~/.hermes`，且不听从环境变量，可在激活 Landlock 之前建立软链接：

```python
import os

tenant_dir = "/data/tenants/tenant_002"
hermes_home = os.path.expanduser("~/.hermes")

# 如果之前有软链接，先删掉
if os.path.islink(hermes_home):
    os.unlink(hermes_home)

# 建立软链接：把 ~/.hermes 物理指向租户 002 的真实数据夹
os.symlink(tenant_dir, hermes_home)

# 【核心】Landlock 在内核层面会解析软链接背后的真实路径进行安全校验
# 必须把两个路径都加入白名单
ruleset.add_path(path=tenant_dir, allow_read=True, allow_write=True)
ruleset.add_path(path=hermes_home, allow_read=True, allow_write=True)

# 锁死沙箱
ruleset.restrict()
```

> ⚠️ **竞态条件警告**：高并发场景下（多租户同时调用），频繁修改 `~/.hermes` 软链接会引发 Race Condition。仅适合单实例或进程隔离模式。

**方案 C：回到 bwrap（真正的路径映射）**

如果坚决要在内核层面实现类似 Docker 的 `-v /data/tenants/tenant_002:~/.hermes` 路径替换，只能选择 bwrap。bwrap 集成了 Linux mount 命名空间，拥有真正的"路径魔术"能力：

```bash
bwrap --bind /data/tenants/tenant_002 /home/debian_user/.hermes \
      --ro-bind /usr /usr \
      python3 -m hermes_agent
```

#### 架构师建议

| 场景 | 推荐方案 |
|------|----------|
| 高并发运行在同一个 Python 进程（如 FastAPI） | **方案 A**（环境变量 + Landlock），最轻量、零竞态 |
| 单实例/每租户独立进程，需要路径障眼法 | **方案 B**（符号链接 + Landlock） |
| 死磕路径，代码完全不改，需要内核级虚拟文件视图 | **方案 C**（bwrap bind mount） |

---

## 选型决策

| 场景 | 推荐方案 |
|------|----------|
| 需隔离网络（禁止租户联网） | **bwrap**（`--unshare-net`） |
| 需隔离进程列表（隐藏 `ps -ef`） | **bwrap**（`--unshare-pid`） |
| 需彻底干净的虚拟 `/tmp` | **bwrap**（`--tmpfs /tmp`） |
| 只在乎文件不被偷看/串改，追求代码优雅 | **Landlock** |
| 不想折腾 Debian `/usr/lib` 路径映射 | **Landlock** |
| 外层容器无法给 `SYS_ADMIN` 权限 | **Landlock**（Rootless） |

**结论**：两者不互斥。可组合使用——bwrap 做粗粒度隔离（网络/进程/tmp），Landlock 做细粒度文件权限控制（租户数据互不可见）。

---

## 参考

- [zeroclaw-labs/zeroclaw#5126](https://github.com/zeroclaw-labs/zeroclaw/issues/5126)
- [Debian Python Paths](https://matthew-brett.github.io/pydagogue/debian_python_paths.html)
- [AgentSH Secure Sandbox](https://www.agentsh.org/docs/secure-sandbox/)
- [Landlock(7) man page](https://man7.org/linux/man-pages/man7/landlock.7.html)
- [sandlock-cli (Rust crate)](https://crates.io/crates/sandlock-cli)
- [Python parent module](https://pypi.org/project/parent/)
- [Kernel Landlock Userspace API](https://docs.kernel.org/userspace-api/landlock.html)
- [Bubblewrap issue #713](https://github.com/containers/bubblewrap/issues/713)
