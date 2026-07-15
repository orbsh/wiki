# NixOS 配置

**路径**：`~/Configuration/nixos`（NixOS 26 / unstable，基于 flake）

## 主机

| 主机 | 用途 | 硬件 |
|------|------|------|
| `workstation` | 主桌面（COSMIC + Hyprland）| Lenovo 笔记本，MediaTek MT7925 WiFi |
| `server` | 无头服务器 / K8s 控制节点 | 裸机，仅有线 |
| `qemu` | QEMU/KVM 虚拟机（从 VirtualBox 重命名）| 虚拟 |
| `portable` | 可启动 USB 便携系统（btrfs 子卷）| 移动 SSD，可在任何 x86_64 机器上运行 |
| `k8s-worker` | K8s 工作节点 | 通过 `k8s-role.nix` 数据驱动 |

## 模块结构

```
nixos/
├── flake.nix
├── x.nu                          # Nushell 辅助：mount-btrfs、install、enter、switch
├── libs/
│   └── nixos-builder.nix         # 构建工厂（baseModules 仅含 disko + HM 集成，不含业务模块）
├── hosts/                        # 硬件配置 + 选项值（不碰逻辑）
│   ├── workstation/
│   ├── server/
│   ├── qemu/
│   └── portable/                 # USB 移动系统盘
├── presets/                      # 系统配置预设集（原 roles/，ADR-004）
│   ├── workstation-base.nix      # 显式导入 core.nix + home.nix + desktop/*
│   └── server.nix                # 显式导入 core.nix + home.nix
└── modules/
    ├── system/
    │   ├── core.nix              # 纯系统模块（sys, base, nix, users, network, container, extra）
    │   ├── home.nix              # HM 聚合入口（与 core.nix 解耦，ADR-011）
    │   └── units/
    │       ├── home-base.nix     # 通用 HM（stateVersion）
    │       ├── home-shell.nix    # 通用 HM（nushell）
    │       ├── home-editors.nix  # 通用 HM（helix/neovim）
    │       ├── home-git.nix      # 通用 HM（git）
    │       └── ...               # sys.nix, base.nix, nix.nix, network.nix 等
    ├── desktop/
    │   ├── full.nix              # 桌面系统模块
    │   ├── home.nix              # 桌面 HM 聚合
    │   └── units/
    │       ├── home-terminals.nix  # 桌面 HM（ghostty/alacritty/zellij）
    │       ├── home-xdg.nix       # 桌面 HM（mimeApps/userDirs）
    │       ├── hyprland.nix       # 跨层混合模块（NixOS + HM 共享变量）
    │       └── ...
    ├── k8s/
    │   ├── k8s-libs.nix          # K8s 节点构建工具
    │   ├── k8s-common.nix        # K8s 基础（kubelet/certs）
    │   ├── k8s-control.nix       # Control plane
    │   ├── k8s-worker.nix        # Worker nodes
    │   ├── k8s-addons.nix        # Addons（Flannel/CoreDNS）
    │   ├── containerd.nix        # Containerd runtime
    │   ├── crio.nix              # CRI-O runtime
    │   ├── coredns.nix           # CoreDNS
    │   ├── cert-manager.nix      # Cert-Manager
    │   ├── envoy-gateway.nix     # Envoy Gateway
    │   ├── istio-gateway.nix     # Istio
    │   └── assets/               # YAML manifests, scripts
    ├── flake-srv/
    │   └── harmonia.nix          # Binary cache 服务
    ├── services/
    │   └── numa.nix              # Numa 本地 DNS + 反向代理（workstation 专用）
    └── dev/                      # 开发工具链
```

### 模块分类规则（ADR-011）

| 类型 | 模式 | 示例 |
|------|------|------|
| 纯系统 | 无 `home-manager.users` 块 | `sys.nix`, `nix.nix`, `network.nix` |
| 纯 HM | 包裹在 `home-manager.users.${user}` 中 | `home-shell.nix`, `home-git.nix` |
| 跨层混合 | NixOS + HM 共享变量 | `hyprland.nix`, `eww.nix`, `rime.nix` |

## 从宿主机更新便携系统

Portable 是移动 SSD 上的可启动 NixOS。从宿主系统更新：

```nu
# 1. 挂载便携驱动器（btrfs 子卷：@, @home, @var, @swap）
use x.nu portable
portable mount-btrfs

# 2. 更新
portable switch          # 默认 portable 主机

# 可选：进入 chroot 调试
portable enter
```

`switch` 直接从宿主机调用 `nixos-rebuild --root /mnt`——无需 chroot。

## WiFi 配置（仅 workstation + portable）

workstation 和 portable 都使用 MediaTek MT7925（WiFi 7）。在每个主机的 `default.nix` 中配置（不在 `common/sys.nix` 中，避免在 server 上安装 iwd）：

- `networking.networkmanager.wifi.backend = "iwd"` — iwd 后端而非 wpa_supplicant
- `networking.networkmanager.wifi.powersave = false` — 禁用省电

## 约定

- 文档和源代码注释只用英文
- WiFi 设置按主机，不在公共模块中
- Server 无 WiFi 配置

## Portable btrfs 子卷布局

| 子卷 | 挂载点 | 用途 |
|------|--------|------|
| `@` | `/` | 根文件系统 |
| `@boot` | `/boot` | EFI 系统分区 |
| `@home` | `/home` | 用户数据 |
| `@var` | `/var` | 系统状态 |
| `@swap` | `/swap` | Swap 文件 |

## 陷阱

- `common/sys.nix` 也被 server 导入——WiFi 特定设置（iwd）不能放在那里
- Portable hardware-configuration 有广泛的 initrd 模块（SATA、NVMe、USB、VirtIO、Thunderbolt）用于在任何主机上启动
- 永远不要自动提交 NixOS 配置变更
- `services.kubernetes.flannel.enable = true` 创建冲突的 `mynet` 网桥——始终使用 `lib.mkForce false` 并通过 addons 管理 Flannel
- containerd 需要 `cni.conf_dir` 和 `cni.bin_dir`，否则回退到内置 `mynet` CNI
- `clusterDns` 需要列表：`[ "10.0.0.254" ]` 而非字符串
- Nix `''...''` 字符串中的 Bash 变量必须转义：`''${VAR}`
- `--type=merge` 补丁替换整个数组——对 CoreDNS 使用 Strategic Merge 配合 `$setElementOrder/containers`

---

## K8s Ansible → NixOS 迁移

**日期**：2025-05-18 | **来源**：`~/Configuration/nixos/docs/memos/k8s-ansible-to-nixos-migration.md`

### 为何从 Ansible 迁移

| 痛点 | Ansible | NixOS |
|------|---------|-------|
| 可重现性 | 命令式，依赖执行历史 | 声明式，相同配置始终产生相同结果 |
| 回滚 | 手动 | 一键（GRUB 上一代）|
| 原子升级 | 无 | `switch-to-configuration` |
| 依赖隔离 | 全局安装 | `/nix/store` 隔离 |
| 构建时验证 | 无 | `nix build` 检查语法 + 依赖 |
| 证书管理 | 手动脚本 | easyCerts 自动生成 |

### 架构哲学

声明式不可变基础设施下沉到操作系统层：

```
Git Commit（配置哈希）
  └── NixOS 系统配置
        ├── 内核参数 / sysctl
        ├── systemd 服务（kubelet、kube-proxy、containerd）
        ├── CNI 插件（Flannel cni0、10-flannel.conflist）
        ├── TLS 证书（easyCerts 自动生成）
        └── K8s Addons（Flannel DS、CoreDNS Deployment）
              └── K8s 集群状态（Service、Deployment、Gateway）
```

单个 Git commit 哈希完全重现从操作系统到应用的整个节点状态。

### 项目结构

```
nixos/
├── flake.nix                    # 入口，定义所有主机配置
├── config/
│   ├── nodes.nix                # 集群节点定义（IP/角色/运行时）
│   └── nodes/
│       ├── dev.nix              # 开发集群（combo 节点）
│       ├── small-cluster.nix    # 小集群示例
│       └── large-cluster.nix    # 大集群示例
├── hosts/                       # 主机硬件配置
├── modules/
│   ├── server/
│   │   ├── k8s-common.nix       # K8s 基础（kubelet/certs/禁用 flannel）
│   │   ├── k8s-addons.nix       # K8s Addons（Flannel/CoreDNS/RBAC 声明式部署）
│   │   ├── k8s-control.nix      # Control plane（apiserver/scheduler/controllerManager）
│   │   ├── k8s-worker.nix       # Worker nodes
│   │   ├── k8s-lib.nix          # 节点构建器工具函数
│   │   ├── istio-gateway.nix    # Istio + Gateway API（带清理服务）
│   │   ├── cert-manager.nix     # Cert-Manager + Issuers
│   │   ├── crio.nix             # CRI-O runtime
│   │   └── containerd.nix       # Containerd runtime
│   └── common/                  # 共享配置
└── home/                        # Home Manager 配置
```

### 节点角色

- **control** — 仅控制平面（apiserver、scheduler、controllerManager、etcd）
- **worker** — 仅 kubelet + kube-proxy，调度常规 Pod
- **combo** — 控制平面和工作节点（用于小集群）

### 部署命令

```bash
nixos-rebuild switch \
  --flake .#dev__dxserver \
  --target-host root@dxserver \
  --build-host root@dxserver
```

### 问题与解决方案

#### 1. kube-proxy 未启用

- **症状**：ClusterIP 不可达，DNS 解析失败，NodePort 不通
- **原因**：`services.kubernetes.proxy.enable` 需要 easyCerts 或手动证书。无 easyCerts 时，kube-proxy 客户端证书未生成，systemd 服务失败
- **修复**：启用 `services.kubernetes.easyCerts = true`

#### 2. kubelet DNS 配置缺失

- **症状**：Pod 无法解析 `*.svc.cluster.local`
- **修复**：添加 `clusterDns = [ "10.0.0.254" ]; clusterDomain = "cluster.local";`

#### 3. 外部资源运行时下载超时

- **症状**：`deploy-gateway-api-crds.service` / Flannel manifest 从 GitHub 下载超时
- **修复**：构建时使用 `pkgs.fetchurl`，运行时从 `/nix/store` 读取

#### 3.5. Flannel 部署（重构到 k8s-addons.nix）

- **旧**：`kube-flannel-apply.service`（独立 systemd oneshot）
- **新**：`k8s-addons-apply.service`（统一 K8s Addons 管理）
- **改进**：API Server 就绪等待（12×10s 重试），自动删除冲突的 ClusterRoleBindings，重建前自动删除旧 DaemonSet，Server-Side Apply，CoreDNS strategic merge 配合 `$setElementOrder/containers`
- **关键修复——双网桥冲突**：k8s-common.nix 中 `services.kubernetes.flannel.enable = lib.mkForce false`
- **关键修复——containerd CNI 发现**：添加 `cni.conf_dir = "/etc/cni/net.d"; cni.bin_dir = "/opt/cni/bin";`
- **关键修复——部署顺序**：等待 `kubectl rollout status daemonset kube-flannel-ds` 然后 `ip link show cni0` 再 CoreDNS 补丁
- **关键修复——API Server TLS**：添加 cni0 网桥 IP `"10.1.1.1"` 到 `extraSANs`，然后删除旧证书：`sudo rm -f /var/lib/kubernetes/secrets/kube-apiserver*.pem && nixos-rebuild switch`

#### 3.6. istio-system 删除卡住

- **症状**：`kubectl delete namespace istio-system` 卡在 Terminating
- **原因**：Istio Gateway 和 IstioOperator CR finalizers 阻止删除
- **修复**：`cleanup-istio.service` — 剥离所有资源的 finalizers，强制删除命名空间

#### 4. CoreDNS targetPort 错误

- **症状**：Pod DNS 超时，iptables 指向端口 10053
- **修复**：补丁 kube-dns Service：`targetPort = 53`

#### 5. kube-proxy DaemonSet 镜像硬编码

- **修复**：使用 `${lib.getVersion pkgs.kubernetes}` 自动跟随系统版本；最终通过 easyCerts 完全移除

#### 6. kubectl 硬编码证书路径

- **修复**：使用 easyCerts 时，使用 `--kubeconfig /etc/kubernetes/cluster-admin.kubeconfig`

#### 8. systemd 服务在 API Server 就绪前运行

- **修复**：所有依赖 K8s API 的 systemd 服务需要 API Server 等待循环：
```bash
for i in $(seq 1 12); do
  kubectl cluster-info --request-timeout=5s >/dev/null 2>&1 && break
  sleep 10
done
```

#### 11. containerd 内置回退 CNI

- **症状**：`mynet` 网桥持续存在，Pod 无网络
- **修复**：在 containerd 设置中配置 `cni.conf_dir` 和 `cni.bin_dir`。修复后，重启 containerd 并删除现有 Pod。

#### 12. CoreDNS 到 API Server 的 TLS 连接失败

- **症状**：`x509: certificate is valid for ..., not 10.1.1.1`
- **修复**：在 k8s-common.nix 的 `extraSANs` 中添加 `"10.1.1.1"`，重启 kube-apiserver

#### 14. CoreDNS 补丁覆盖容器镜像字段

- **修复**：使用 Strategic Merge（默认）配合 `$setElementOrder/containers`，而非 `--type=merge`

#### 15. Envoy Gateway localhost:80 连接拒绝

- **根因**：(1) xDS Listener Resources 为空，(2) Gitea Service selector 也匹配 gitea-db pods，(3) EnvoyProxy targetPort 不匹配
- **端口映射**：Gateway 端口 80 → xDS listener 10080，443 → 10443，22 → 10022，53 → 10053

### 回滚

```bash
nix-env --list-generations --profile /nix/var/nix/profiles/system
nix-env --rollback --profile /nix/var/nix/profiles/system
# 或从 GRUB 菜单选择上一代
```

---

## 架构决策记录（ADR）

### ADR-001: Overlay 启用策略 — 按域而非按配置值

**日期**: 2026-06-01  
**上下文**: `libs/nixos-builder.nix` 第 27 行

**问题**: 如何决定哪些节点应用 `modules/overlay/` 下的自定义 overlay（如 nushell 0.113.0）？

**决策**: 采用按域判断：`nodeAttrs.domainName == "workstations"` 决定是否启用 overlay。

**理由**:
1. **扩展合理** — workstations 是开发机，应用全部 overlay 符合直觉；其他域（k8s/server/portable）用 nixpkgs 官方包
2. **改动最小** — 只改 builder 一行，不需要在 flake.nix 中传递额外配置
3. **职责清晰** — builder 层决定「哪些机器需要自定义包」，role 层只声明「这个角色的功能需求」，两层不互相渗透
4. **无间接依赖** — 不需要在 flake 层提前导入 role 模块

**后果**: 新增需要 overlay 的域时，需在 builder 中添加对应的域名判断，可改为白名单模式 `builtins.elem domainName [ "workstations" "other" ]`

### ADR-002: Nushell 版本与 Develop Mode 是两个独立维度

**日期**: 2026-06-01

**问题**: nushell 的「版本」（overlay 控制）和「配置部署方式」（developMode 控制）容易混淆。

**概念区分**:

| 维度 | Overlay（版本）| Develop Mode（配置部署）|
|------|---------------|----------------------|
| **控制什么** | nushell 二进制版本（0.113.0 vs 0.112.2）| nushell 配置文件从哪来 |
| **在哪设置** | `modules/overlay/nushell.nix` | `programs.nushell.developMode` in role 模块 |
| **生效范围** | 系统级 `pkgs.nushell` | Home Manager 用户配置 |

**组合策略**:
- **workstations** → overlay + developMode=true（最新版 + 本地配置实时编辑）
- **k8s/server/portable/ISO** → 官方 + developMode=false（稳定版本 + 固化配置）

### ADR-003: 系统配置画像目录命名 — `presets/` 而非 `roles/`

**日期**: 2026-06-01

**问题**: `modules/roles/` 目录（workstation-base.nix、server.nix 等）与 K8s 节点的 `role = "combo"` 同名但含义不同，容易混淆。

**决策**: `modules/roles/` → `modules/presets/`，K8s 的 `role` 保持不变。

**理由**:
1. **消除歧义** — `presets/`（系统配置预设）与 `role = "combo"`（K8s 节点功能）不再同名
2. **层级一致** — 与 `desktop/` 下的预设（mini.nix / base.nix / full.nix）概念统一
3. **避免人格化** — `profiles` 暗指"用户画像"，`presets` 更中性，只表达"一组预设配置"

**后果**: 所有 import 路径从 `../../modules/roles/` 改为 `../../modules/presets/`

### ADR-004: SurrealDB 版本管理 — 模块级局部 Overlay

**日期**: 2026-06-02

**问题**: SurrealDB 需要固定到特定版本，如何选择版本覆盖方式？

**决策**: 在模块内部通过 `nixpkgs.overlays` 注入 overlay（模块级 overlay）。

**理由**:
1. **引入即启用** — 模块职责内聚：声明启用服务 + 统一版本，一次 import 搞定
2. **版本安全** — 防止同 host 内意外混用不同版本（`overrideAttrs` 方案存在此风险）
3. **不影响其他 host** — 仅引入该模块的 host 生效，不污染全局 overlay

### ADR-005: Harmonia 二进制缓存引入策略

**日期**: 2026-06-05

**决策**: 分层引入——Server preset 全加（自动覆盖 K8s）、Workstation preset 全加、Portable 单独导入、QEMU 不加。

| 主机/集群 | Harmonia | 来源 |
|-----------|----------|------|
| workstations | ✅ | `presets/workstation-base.nix` |
| server + k8s | ✅ | `presets/server.nix` → `k8s-libs.nix` |
| portable | ✅ | 节点直接导入 |
| qemu | ❌ | 不需要 |

### ADR-006: /tmp tmpfs 挂载策略

**日期**: 2026-06-05

**问题**: `/tmp` 挂载到 tmpfs 可避免 SSD 磨损，但会带来突发性内存争抢风险。

**决策**: 按主机类型分层——

| 主机类型 | tmpfs | 理由 |
|----------|-------|------|
| workstations | ✅ `50%` | 内存充沛，频繁构建，SSD 寿命敏感 |
| server / k8s | ❌ | 稳定性优先，企业级 SSD 无寿命焦虑 |
| portable / qemu | ❌ | 内存有限 |

**Server 折中**: 若内存 64G+ 且有大量结余，可用固定值 `tmpfsSize = "16G"` + 充足 swap。

### ADR-007: Harmonia 客户端使用 CLI 参数配置 substituter

**日期**: 2026-06-09

**问题**: NixOS 配置中的 `substituters` 需 switch 后才生效，但 switch 本身就需要用它加速构建（自举问题）。

**决策**: 使用命令行参数，不在 NixOS 配置中持久化 harmonia 地址。

```bash
sudo nixos-rebuild switch --flake .#<host> \
  --option substituters 'http://localhost:5101 https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store https://cache.nixos.org' \
  --option extra-trusted-public-keys 'harmonia-local:bF/+RpECJWbbE8W7/hu1jWRlkQqu/+cXoVrWFENmqXY='
```

**关键点**: `--option substituters`（替换整个列表，harmonia 放首位）而非 `--option extra-substituters`（追加到末尾）。

### ADR-008: Portable USB 系统盘禁用 systemd initrd

**日期**: 2026-06-10

**问题**: Portable 作为 USB 移动系统盘，启用 systemd initrd 后多个服务无限等待设备就绪。

**根因**: USB 设备枚举慢，systemd initrd 通过 udev 事件驱动设备发现，可能错过设备就绪信号，导致 `.device` unit 超时阻塞启动。stage-1 init 的阻塞等待天然适配慢速 USB 设备。

**决策**: `boot.initrd.systemd.enable = false`（显式禁用）。

**历史**: commit `44da8c8` 启用 systemd initrd 导致问题复发，回退后解决。固定磁盘(NVMe/SATA)无此问题。

### ADR-009: 模块组织与 Home Manager 集成策略

**日期**: 2026-06-10

**问题**: `modules/home/` 目录混合了桌面 HM 和通用 HM；`core.nix` 隐式导入 `home.nix`；builder 硬编码业务模块。

**决策**（方案 D）:
1. **HM 按归属域组织** — 通用 HM → `system/units/home-*.nix`；桌面 HM → `desktop/units/home-*.nix`；`modules/home/` 删除
2. **core.nix 与 home.nix 解耦** — core.nix 纯系统，home.nix 独立 HM 聚合入口
3. **Presets 显式导入** — preset 声明依赖（core + home + desktop），builder 不含业务模块
4. **跨层模块保持混合** — hyprland/eww/rime 同时需 NixOS+HM 配置且共享变量，不拆分

### ADR-010: K8s DNS 架构 — 分层解析 + 全局公共 DNS

**日期**: 2026-06-11

**问题**: NixOS 宿主机 `/etc/resolv.conf` 指向 `127.0.0.1`（本地 CoreDNS stub resolver），但 K8s pod 内 `127.0.0.1` 是容器 loopback（无 DNS 服务），导致集群内 CoreDNS `forward . /etc/resolv.conf` 失败。

**根因链路**:
```
Pod 查询外部域名
  → kube-dns ClusterIP (10.0.0.254)
  → 集群内 CoreDNS pod（dnsPolicy=Default，使用宿主机 /etc/resolv.conf）
  → Corefile: forward . /etc/resolv.conf
  → pod 内 /etc/resolv.conf 指向 127.0.0.1（容器 loopback，无 DNS）
  → 解析失败
```

**决策**: 分层解析 + 全局公共 DNS 配置

1. **公共 DNS 作为全局配置** — `flake.nix` 的 `commonArgs.publicDnsServers`（地理位置相关，中国大陆：`223.5.5.5`、`119.29.29.29`、`1.1.1.1`）
2. **宿主机 CoreDNS 引入即启用** — `modules/services/coredns.nix` 设置 `networking.nameservers = [ "127.0.0.1" ]`
3. **kubelet resolv.conf 动态决定** — `modules/k8s/k8s-common.nix`：
   - 有宿主机 CoreDNS → `nameserver <cni0IP>`（pod 通过 cni0 网桥访问宿主机 CoreDNS）
   - 无宿主机 CoreDNS → `nameserver <公共DNS>`
4. **集群内 CoreDNS Corefile 动态决定** — `modules/k8s/assets/patch-coredns.sh`：
   - 有宿主机 CoreDNS → `forward . <cni0IP>`
   - 无宿主机 CoreDNS → `forward . <公共DNS>`

**DNS 链路（有宿主机 CoreDNS）**:
```
Pod 查询外部域名
  → kube-dns ClusterIP (10.0.0.254)
  → 集群内 CoreDNS pod
  → Corefile: forward . <cni0IP>
  → 宿主机 CoreDNS（监听 0.0.0.0:53，通过 cni0 可达）
  → 宿主机 CoreDNS forward 到上游 DNS（阿里 223.5.5.5 等）
```

**理由**:
1. **单一配置源** — 公共 DNS 在 `flake.nix` 定义一次，所有模块通过 `commonArgs` 获取
2. **引入即启用** — 宿主机 CoreDNS 模块自动设置系统 DNS 指向 `127.0.0.1`，无需额外开关
3. **条件适配** — 有/无宿主机 CoreDNS 两种场景自动适配，无需手动配置
4. **地理位置解耦** — 公共 DNS 列表与代码逻辑分离，换地区只改一处

**后果**:
- `libs/nixos-builder.nix` 不再硬编码 `networking.nameservers`（由模块自己决定）
- `modules/k8s/k8s-addons.nix` 和 `modules/k8s/k8s-common.nix` 通过函数参数获取 `publicDnsServers`
- 新增 K8s 节点自动继承 DNS 配置，无需手动设置

### ADR-011: Workstation 本地 DNS — Numa 替代 CoreDNS

**日期**: 2026-07-04

**问题**: Server 运行 K8s，宿主机 CoreDNS 监听 `0.0.0.0:53` 供 pod 通过 cni0 网桥访问（ADR-010）。Workstation 不跑 K8s，不需要宿主机 CoreDNS 的 K8s 转发链路。Numa 提供本地 DNS 解析 + 反向代理，更适合 workstation 场景。两者都要绑定 53 端口，不能同时运行。

**决策**: 按主机类型分流——

| 主机类型 | 本地 DNS | 理由 |
|----------|---------|------|
| server / k8s | CoreDNS（`modules/services/coredns.nix`）| K8s pod 通过 cni0 网桥访问宿主机 DNS，CoreDNS 的 `forward . <cni0IP>` 链路已验证（ADR-010） |
| workstation | Numa | 无 K8s 依赖，Numa 提供 `.numa` TLD 本地域名 + 反向代理，开发更方便 |

**Numa 不与 K8s 冲突**：Numa 只装在 workstation 上，server 保留 CoreDNS。两者是不同主机上的独立服务，不共享 53 端口。

**NixOS 配置**（`modules/services/numa.nix`）：

```nix
{ config, pkgs, lib, ... }:
let
  numaConfigFile = pkgs.writeText "numa.toml" ''
    [dns]
    tld = "numa"
    bind_addr = "127.0.0.1:53"

    [proxy]
    bind_addr = "127.0.0.1"

    [[services]]
    name = "frontend"
    target_port = 5173
  '';
in {
  systemd.services.numa = {
    description = "Numa - Local DNS resolver and reverse proxy";
    after = [ "network.target" ];
    wantedBy = [ "multi-user.target" ];
    serviceConfig = {
      Type = "simple";
      ExecStart = "${pkgs.numa}/bin/numa --config ${numaConfigFile}";
      Restart = "on-failure";
      RestartSec = "5s";
      AmbientCapabilities = [ "CAP_NET_BIND_SERVICE" ];
      CapabilityBoundingSet = [ "CAP_NET_BIND_SERVICE" ];
    };
  };

  networking.nameservers = [ "127.0.0.1" ];

  services.resolved = {
    enable = true;
    extraConfig = ''
      [Resolve]
      DNS=127.0.0.1
      FallbackDNS=223.5.5.5
      DNSStubListener=no
    '';
  };

  networking.firewall = {
    enable = true;
    allowedTCPPorts = [ 80 443 5380 853 ];
    allowedUDPPorts = [ 53 443 ];
  };
}
```

**关键配置点**：

- `DNSStubListener=no` — 必须关闭，否则 systemd-resolved 抢占 53 端口与 Numa 冲突
- `AmbientCapabilities = [ "CAP_NET_BIND_SERVICE" ]` — Numa 绑定特权端口（53/80/443/853）需要此 capability
- `pkgs.writeText` — 配置文件存入只读 `/nix/store`，适合纯声明式管理
- 如果依赖 Numa REST API 动态更新配置，`ExecStart` 改为指向可写路径 `/var/lib/numa/numa.toml`

**引入方式**：在 `presets/workstation-base.nix` 中导入 `modules/services/numa.nix`，server preset 不导入。
