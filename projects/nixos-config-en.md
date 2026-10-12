# NixOS Configuration

> **Languages:** [English](nixos-config-en.md) (primary) · [中文](nixos-config.md)

**Path**: `~/Configuration/nixos` (NixOS 26 / unstable, flake-based)

## Hosts

| Host | Purpose | Hardware |
|------|------|------|
| `workstation` | Primary desktop (COSMIC + Hyprland)| Lenovo laptop, MediaTek MT7925 WiFi |
| `server` | Headless server / K8s control node | Bare metal, wired only |
| `qemu` | QEMU/KVM virtual machine (renamed from VirtualBox)| Virtual |
| `portable` | Bootable USB portable system (btrfs subvolumes)| Mobile SSD, runs on any x86_64 machine |
| `k8s-worker` | K8s worker node | Data-driven via `k8s-role.nix` |

## Module Structure

```
nixos/
├── flake.nix
├── x.nu                          # Nushell helper: mount-btrfs, install, enter, switch
├── libs/
│   └── nixos-builder.nix         # Build factory (baseModules contains only disko + HM integration, no business modules)
├── hosts/                        # Hardware configuration + option values (does not touch logic)
│   ├── workstation/
│   ├── server/
│   ├── qemu/
│   └── portable/                 # USB mobile system disk
├── presets/                      # System configuration preset sets (formerly roles/, ADR-004)
│   ├── workstation-base.nix      # Explicitly imports core.nix + home.nix + desktop/*
│   └── server.nix                # Explicitly imports core.nix + home.nix
└── modules/
    ├── system/
    │   ├── core.nix              # Pure system module (sys, base, nix, users, network, container, extra)
    │   ├── home.nix              # HM aggregation entry (decoupled from core.nix, ADR-011)
    │   └── units/
    │       ├── home-base.nix     # Common HM (stateVersion)
    │       ├── home-shell.nix    # Common HM (nushell)
    │       ├── home-editors.nix  # Common HM (helix/neovim)
    │       ├── home-git.nix      # Common HM (git)
    │       └── ...               # sys.nix, base.nix, nix.nix, network.nix etc.
    ├── desktop/
    │   ├── full.nix              # Desktop system module
    │   ├── home.nix              # Desktop HM aggregation
    │   └── units/
    │       ├── home-terminals.nix  # Desktop HM (ghostty/alacritty/zellij)
    │       ├── home-xdg.nix       # Desktop HM (mimeApps/userDirs)
    │       ├── hyprland.nix       # Cross-layer hybrid module (NixOS + HM shared variables)
    │       └── ...
    ├── k8s/
    │   ├── k8s-libs.nix          # K8s node build tools
    │   ├── k8s-common.nix        # K8s foundation (kubelet/certs)
    │   ├── k8s-control.nix       # Control plane
    │   ├── k8s-worker.nix        # Worker nodes
    │   ├── k8s-addons.nix        # Addons (Flannel/CoreDNS)
    │   ├── containerd.nix        # Containerd runtime
    │   ├── crio.nix              # CRI-O runtime
    │   ├── coredns.nix           # CoreDNS
    │   ├── cert-manager.nix      # Cert-Manager
    │   ├── envoy-gateway.nix     # Envoy Gateway
    │   ├── istio-gateway.nix     # Istio
    │   └── assets/               # YAML manifests, scripts
    ├── flake-srv/
    │   └── harmonia.nix          # Binary cache service
    ├── services/
    │   └── numa.nix              # Numa local DNS + reverse proxy (workstation only)
    └── dev/                      # Development toolchain
```

### Module Classification Rules (ADR-011)

| Type | Pattern | Example |
|------|------|------|
| Pure system | No `home-manager.users` block | `sys.nix`, `nix.nix`, `network.nix` |
| Pure HM | Wrapped in `home-manager.users.${user}` | `home-shell.nix`, `home-git.nix` |
| Cross-layer hybrid | NixOS + HM shared variables | `hyprland.nix`, `eww.nix`, `rime.nix` |

## Updating the Portable System from the Host

Portable is a bootable NixOS on a mobile SSD. To update from the host system:

```nu
# 1. Mount the portable drive (btrfs subvolumes: @, @home, @var, @swap)
use x.nu portable
portable mount-btrfs

# 2. Update
portable switch          # default portable host

# Optional: enter chroot for debugging
portable enter
```

`switch` calls `nixos-rebuild --root /mnt` directly from the host—no chroot needed.

## WiFi Configuration (workstation + portable only)

Both workstation and portable use the MediaTek MT7925 (WiFi 7). Configure this in each host's `default.nix` (not in `common/sys.nix`, to avoid installing iwd on server):

- `networking.networkmanager.wifi.backend = "iwd"` — iwd backend instead of wpa_supplicant
- `networking.networkmanager.wifi.powersave = false` — disable power saving

## Conventions

- Documentation and source code comments use English only
- WiFi settings are per-host, not in common modules
- Server has no WiFi configuration

## Portable btrfs Subvolume Layout

| Subvolume | Mount point | Purpose |
|------|--------|------|
| `@` | `/` | Root filesystem |
| `@boot` | `/boot` | EFI system partition |
| `@home` | `/home` | User data |
| `@var` | `/var` | System state |
| `@swap` | `/swap` | Swap file |

## Pitfalls

- `common/sys.nix` is also imported by server—WiFi-specific settings (iwd) cannot go there
- Portable hardware-configuration has broad initrd modules (SATA, NVMe, USB, VirtIO, Thunderbolt) for booting on any host
- Never auto-commit NixOS configuration changes
- `services.kubernetes.flannel.enable = true` creates a conflicting `mynet` bridge—always use `lib.mkForce false` and manage Flannel via addons
- containerd requires `cni.conf_dir` and `cni.bin_dir`, otherwise it falls back to the built-in `mynet` CNI
- `clusterDns` requires a list: `[ "10.0.0.254" ]` rather than a string
- Bash variables inside Nix `''...''` strings must be escaped: `''${VAR}`
- `--type=merge` patches replace the entire array—use Strategic Merge with `$setElementOrder/containers` for CoreDNS

---

## K8s Ansible → NixOS Migration

**Date**: 2025-05-18 | **Source**: `~/Configuration/nixos/docs/memos/k8s-ansible-to-nixos-migration.md`

### Why Migrate from Ansible

| Pain point | Ansible | NixOS |
|------|---------|-------|
| Reproducibility | Imperative, depends on execution history | Declarative, the same configuration always produces the same result |
| Rollback | Manual | One key (previous generation in GRUB)|
| Atomic upgrade | None | `switch-to-configuration` |
| Dependency isolation | Global installation | `/nix/store` isolation |
| Build-time validation | None | `nix build` checks syntax + dependencies |
| Certificate management | Manual scripts | easyCerts auto-generation |

### Architecture Philosophy

Declarative immutable infrastructure sinks down to the operating system layer:

```
Git Commit (configuration hash)
  └── NixOS system configuration
        ├── Kernel parameters / sysctl
        ├── systemd services (kubelet, kube-proxy, containerd)
        ├── CNI plugins (Flannel cni0, 10-flannel.conflist)
        ├── TLS certificates (easyCerts auto-generated)
        └── K8s Addons (Flannel DS, CoreDNS Deployment)
              └── K8s cluster state (Service, Deployment, Gateway)
```

A single Git commit hash fully reproduces the entire node state from the operating system to the application.

### Project Structure

```
nixos/
├── flake.nix                    # Entry point, defines all host configurations
├── config/
│   ├── nodes.nix                # Cluster node definitions (IP/role/runtime)
│   └── nodes/
│       ├── dev.nix              # Development cluster (combo node)
│       ├── small-cluster.nix    # Small cluster example
│       └── large-cluster.nix    # Large cluster example
├── hosts/                       # Host hardware configuration
├── modules/
│   ├── server/
│   │   ├── k8s-common.nix       # K8s foundation (kubelet/certs/disable flannel)
│   │   ├── k8s-addons.nix       # K8s Addons (Flannel/CoreDNS/RBAC declarative deployment)
│   │   ├── k8s-control.nix      # Control plane (apiserver/scheduler/controllerManager)
│   │   ├── k8s-worker.nix       # Worker nodes
│   │   ├── k8s-lib.nix          # Node builder utility functions
│   │   ├── istio-gateway.nix    # Istio + Gateway API (with cleanup service)
│   │   ├── cert-manager.nix     # Cert-Manager + Issuers
│   │   ├── crio.nix             # CRI-O runtime
│   │   └── containerd.nix       # Containerd runtime
│   └── common/                  # Shared configuration
└── home/                        # Home Manager configuration
```

### Node Roles

- **control** — Control plane only (apiserver, scheduler, controllerManager, etcd)
- **worker** — kubelet + kube-proxy only, schedules regular Pods
- **combo** — Control plane and worker node (for small clusters)

### Deployment Commands

```bash
nixos-rebuild switch \
  --flake .#dev__dxserver \
  --target-host root@dxserver \
  --build-host root@dxserver
```

### Problems and Solutions

#### 1. kube-proxy not enabled

- **Symptom**: ClusterIP unreachable, DNS resolution fails, NodePort not working
- **Cause**: `services.kubernetes.proxy.enable` requires easyCerts or manual certificates. Without easyCerts, the kube-proxy client certificate is not generated and the systemd service fails
- **Fix**: Enable `services.kubernetes.easyCerts = true`

#### 2. Missing kubelet DNS configuration

- **Symptom**: Pods cannot resolve `*.svc.cluster.local`
- **Fix**: Add `clusterDns = [ "10.0.0.254" ]; clusterDomain = "cluster.local";`

#### 3. Runtime download timeout for external resources

- **Symptom**: `deploy-gateway-api-crds.service` / Flannel manifest download from GitHub times out
- **Fix**: Use `pkgs.fetchurl` at build time, read from `/nix/store` at runtime

#### 3.5. Flannel deployment (refactored into k8s-addons.nix)

- **Old**: `kube-flannel-apply.service` (standalone systemd oneshot)
- **New**: `k8s-addons-apply.service` (unified K8s Addons management)
- **Improvements**: API Server readiness wait (12×10s retry), automatic deletion of conflicting ClusterRoleBindings, automatic deletion of the old DaemonSet before re-creation, Server-Side Apply, CoreDNS strategic merge with `$setElementOrder/containers`
- **Key fix—dual bridge conflict**: `services.kubernetes.flannel.enable = lib.mkForce false` in k8s-common.nix
- **Key fix—containerd CNI discovery**: Add `cni.conf_dir = "/etc/cni/net.d"; cni.bin_dir = "/opt/cni/bin";`
- **Key fix—deployment order**: Wait for `kubectl rollout status daemonset kube-flannel-ds` then `ip link show cni0` before the CoreDNS patch
- **Key fix—API Server TLS**: Add the cni0 bridge IP `"10.1.1.1"` to `extraSANs`, then delete the old certificates: `sudo rm -f /var/lib/kubernetes/secrets/kube-apiserver*.pem && nixos-rebuild switch`

#### 3.6. istio-system deletion stuck

- **Symptom**: `kubectl delete namespace istio-system` stuck in Terminating
- **Cause**: Istio Gateway and IstioOperator CR finalizers block deletion
- **Fix**: `cleanup-istio.service` — strips finalizers from all resources, force-deletes the namespace

#### 4. CoreDNS targetPort error

- **Symptom**: Pod DNS times out, iptables points to port 10053
- **Fix**: Patch the kube-dns Service: `targetPort = 53`

#### 5. kube-proxy DaemonSet image hardcoded

- **Fix**: Use `${lib.getVersion pkgs.kubernetes}` to automatically follow the system version; finally removed entirely via easyCerts

#### 6. kubectl hardcoded certificate paths

- **Fix**: When using easyCerts, use `--kubeconfig /etc/kubernetes/cluster-admin.kubeconfig`

#### 8. systemd services run before the API Server is ready

- **Fix**: All systemd services that depend on the K8s API need an API Server wait loop:
```bash
for i in $(seq 1 12); do
  kubectl cluster-info --request-timeout=5s >/dev/null 2>&1 && break
  sleep 10
done
```

#### 11. containerd built-in fallback CNI

- **Symptom**: The `mynet` bridge persists, Pods have no network
- **Fix**: Configure `cni.conf_dir` and `cni.bin_dir` in the containerd settings. After fixing, restart containerd and delete existing Pods.

#### 12. CoreDNS TLS connection to the API Server fails

- **Symptom**: `x509: certificate is valid for ..., not 10.1.1.1`
- **Fix**: Add `"10.1.1.1"` to `extraSANs` in k8s-common.nix, restart kube-apiserver

#### 14. CoreDNS patch overwrites the container image field

- **Fix**: Use Strategic Merge (default) with `$setElementOrder/containers` rather than `--type=merge`

#### 15. Envoy Gateway localhost:80 connection refused

- **Root cause**: (1) xDS Listener Resources empty, (2) the Gitea Service selector also matches gitea-db pods, (3) EnvoyProxy targetPort mismatch
- **Port mapping**: Gateway port 80 → xDS listener 10080, 443 → 10443, 22 → 10022, 53 → 10053

### Rollback

```bash
nix-env --list-generations --profile /nix/var/nix/profiles/system
nix-env --rollback --profile /nix/var/nix/profiles/system
# Or select the previous generation from the GRUB menu
```

---

## Architecture Decision Records (ADR)

### ADR-001: Overlay Enablement Strategy — By Domain, Not By Configuration Value

**Date**: 2026-06-01  
**Context**: `libs/nixos-builder.nix` line 27

**Problem**: How to decide which nodes apply the custom overlays under `modules/overlay/` (e.g. nushell 0.113.0)?

**Decision**: Use a per-domain judgment: `nodeAttrs.domainName == "workstations"` decides whether to enable the overlay.

**Rationale**:
1. **Reasonable extension** — workstations are development machines, so applying all overlays is intuitive; other domains (k8s/server/portable) use official nixpkgs packages
2. **Minimal change** — only one line in the builder changes, no extra configuration needs to be passed in flake.nix
3. **Clear responsibility** — the builder layer decides "which machines need custom packages", the role layer only declares "the functional requirements of this role", and the two layers do not bleed into each other
4. **No indirect dependency** — no need to import role modules ahead of time in the flake layer

**Consequences**: When adding a new domain that needs overlays, the corresponding domain-name check must be added to the builder; this can be changed to a whitelist pattern `builtins.elem domainName [ "workstations" "other" ]`

### ADR-002: Nushell Version and Develop Mode Are Two Independent Dimensions

**Date**: 2026-06-01

**Problem**: nushell's "version" (controlled by overlay) and "how the configuration is deployed" (controlled by developMode) are easily confused.

**Conceptual distinction**:

| Dimension | Overlay (version)| Develop Mode (config deployment)|
|------|---------------|----------------------|
| **Controls what** | nushell binary version (0.113.0 vs 0.112.2)| Where the nushell config files come from |
| **Where set** | `modules/overlay/nushell.nix` | `programs.nushell.developMode` in the role module |
| **Scope** | System-level `pkgs.nushell` | Home Manager user configuration |

**Combination strategy**:
- **workstations** → overlay + developMode=true (latest version + live local config editing)
- **k8s/server/portable/ISO** → official + developMode=false (stable version + pinned configuration)

### ADR-003: System Configuration Profile Directory Naming — `presets/` Rather Than `roles/`

**Date**: 2026-06-01

**Problem**: The `modules/roles/` directory (workstation-base.nix, server.nix, etc.) shares its name with the K8s node `role = "combo"` but means something different, which is easily confused.

**Decision**: `modules/roles/` → `modules/presets/`, the K8s `role` stays unchanged.

**Rationale**:
1. **Eliminate ambiguity** — `presets/` (system configuration presets) and `role = "combo"` (K8s node function) no longer share a name
2. **Consistent levels** — conceptually unified with the presets under `desktop/` (mini.nix / base.nix / full.nix)
3. **Avoid personification** — `profiles` implies "user profile", `presets` is more neutral and only expresses "a set of preset configurations"

**Consequences**: All import paths change from `../../modules/roles/` to `../../modules/presets/`

### ADR-004: SurrealDB Version Management — Module-Level Local Overlay

**Date**: 2026-06-02

**Problem**: SurrealDB needs to be pinned to a specific version; how to choose the version override method?

**Decision**: Inject the overlay inside the module via `nixpkgs.overlays` (module-level overlay).

**Rationale**:
1. **Introduce and it's enabled** — cohesive module responsibility: declare the service enabled + unify the version, done in one import
2. **Version safety** — prevents accidentally mixing different versions within the same host (the `overrideAttrs` approach carries this risk)
3. **No impact on other hosts** — only takes effect on hosts that import the module, does not pollute the global overlay

### ADR-005: Harmonia Binary Cache Introduction Strategy

**Date**: 2026-06-05

**Decision**: Layered introduction—Server preset adds it everywhere (automatically covering K8s), Workstation preset adds it everywhere, Portable imports it separately, QEMU does not add it.

| Host/Cluster | Harmonia | Source |
|-----------|----------|------|
| workstations | ✅ | `presets/workstation-base.nix` |
| server + k8s | ✅ | `presets/server.nix` → `k8s-libs.nix` |
| portable | ✅ | Node imports directly |
| qemu | ❌ | Not needed |

### ADR-006: /tmp tmpfs Mount Strategy

**Date**: 2026-06-05

**Problem**: Mounting `/tmp` on tmpfs avoids SSD wear, but introduces a risk of sudden memory contention.

**Decision**: Layered by host type—

| Host type | tmpfs | Rationale |
|----------|-------|------|
| workstations | ✅ `50%` | Plenty of memory, frequent builds, SSD lifespan sensitive |
| server / k8s | ❌ | Stability first, enterprise-grade SSDs have no lifespan concerns |
| portable / qemu | ❌ | Limited memory |

**Server compromise**: If memory is 64G+ with plenty to spare, a fixed value `tmpfsSize = "16G"` + sufficient swap can be used.

### ADR-007: Harmonia Client Configures substituter via CLI Arguments

**Date**: 2026-06-09

**Problem**: `substituters` in the NixOS configuration only takes effect after switch, but switch itself needs it to speed up builds (bootstrapping problem).

**Decision**: Use command-line arguments; do not persist the harmonia address in the NixOS configuration.

```bash
sudo nixos-rebuild switch --flake .#<host> \
  --option substituters 'http://localhost:5101 https://mirrors.tuna.tsinghua.edu.cn/nix-channels/store https://cache.nixos.org' \
  --option extra-trusted-public-keys 'harmonia-local:bF/+RpECJWbbE8W7/hu1jWRlkQqu/+cXoVrWFENmqXY='
```

**Key point**: `--option substituters` (replaces the entire list, harmonia first) rather than `--option extra-substituters` (appends to the end).

### ADR-008: Disable systemd initrd on the Portable USB System Disk

**Date**: 2026-06-10

**Problem**: With Portable as a USB mobile system disk, enabling systemd initrd causes multiple services to wait indefinitely for devices to become ready.

**Root cause**: USB device enumeration is slow; systemd initrd drives device discovery through udev events and may miss the device-ready signal, causing `.device` units to time out and block boot. The blocking wait of stage-1 init naturally suits slow USB devices.

**Decision**: `boot.initrd.systemd.enable = false` (explicitly disabled).

**History**: Commit `44da8c8` enabled systemd initrd and caused the problem to recur; it was resolved after reverting. Fixed disks (NVMe/SATA) do not have this problem.

### ADR-009: Module Organization and Home Manager Integration Strategy

**Date**: 2026-06-10

**Problem**: The `modules/home/` directory mixes desktop HM and common HM; `core.nix` implicitly imports `home.nix`; the builder hardcodes business modules.

**Decision** (Option D):
1. **Organize HM by belonging domain** — common HM → `system/units/home-*.nix`; desktop HM → `desktop/units/home-*.nix`; `modules/home/` deleted
2. **Decouple core.nix from home.nix** — core.nix is pure system, home.nix is an independent HM aggregation entry
3. **Explicit preset imports** — the preset declares its dependencies (core + home + desktop), the builder contains no business modules
4. **Cross-layer modules stay hybrid** — hyprland/eww/rime simultaneously need NixOS+HM configuration and share variables, so they are not split apart

### ADR-010: K8s DNS Architecture — Layered Resolution + Global Public DNS

**Date**: 2026-06-11

**Problem**: The NixOS host's `/etc/resolv.conf` points to `127.0.0.1` (the local CoreDNS stub resolver), but inside a K8s pod `127.0.0.1` is the container loopback (no DNS service), causing the in-cluster CoreDNS `forward . /etc/resolv.conf` to fail.

**Root cause chain**:
```
Pod queries an external domain name
  → kube-dns ClusterIP (10.0.0.254)
  → in-cluster CoreDNS pod (dnsPolicy=Default, uses host /etc/resolv.conf)
  → Corefile: forward . /etc/resolv.conf
  → /etc/resolv.conf inside the pod points to 127.0.0.1 (container loopback, no DNS)
  → resolution fails
```

**Decision**: Layered resolution + global public DNS configuration

1. **Public DNS as global configuration** — `commonArgs.publicDnsServers` in `flake.nix` (location-dependent; mainland China: `223.5.5.5`, `119.29.29.29`, `1.1.1.1`)
2. **Host CoreDNS enabled on import** — `modules/services/coredns.nix` sets `networking.nameservers = [ "127.0.0.1" ]`
3. **kubelet resolv.conf decided dynamically** — `modules/k8s/k8s-common.nix`:
   - With host CoreDNS → `nameserver <cni0IP>` (the pod reaches the host CoreDNS through the cni0 bridge)
   - Without host CoreDNS → `nameserver <publicDNS>`
4. **In-cluster CoreDNS Corefile decided dynamically** — `modules/k8s/assets/patch-coredns.sh`:
   - With host CoreDNS → `forward . <cni0IP>`
   - Without host CoreDNS → `forward . <publicDNS>`

**DNS chain (with host CoreDNS)**:
```
Pod queries an external domain name
  → kube-dns ClusterIP (10.0.0.254)
  → in-cluster CoreDNS pod
  → Corefile: forward . <cni0IP>
  → host CoreDNS (listening on 0.0.0.0:53, reachable through cni0)
  → host CoreDNS forwards to upstream DNS (AliDNS 223.5.5.5 etc.)
```

**Rationale**:
1. **Single source of configuration** — public DNS is defined once in `flake.nix`, all modules obtain it through `commonArgs`
2. **Enabled on import** — the host CoreDNS module automatically sets the system DNS to point to `127.0.0.1`, no extra switch needed
3. **Conditional adaptation** — automatically adapts to both scenarios (with/without host CoreDNS), no manual configuration needed
4. **Location decoupled** — the public DNS list is separate from the code logic, changing region only requires changing one place

**Consequences**:
- `libs/nixos-builder.nix` no longer hardcodes `networking.nameservers` (the module decides for itself)
- `modules/k8s/k8s-addons.nix` and `modules/k8s/k8s-common.nix` obtain `publicDnsServers` through function parameters
- New K8s nodes automatically inherit the DNS configuration, no manual setup needed

### ADR-011: Workstation Local DNS — Numa Instead of CoreDNS

**Date**: 2026-07-04

**Problem**: Server runs K8s, and the host CoreDNS listens on `0.0.0.0:53` for pods to access through the cni0 bridge (ADR-010). Workstation does not run K8s and does not need the host CoreDNS K8s forwarding chain. Numa provides local DNS resolution + reverse proxy, which is better suited to the workstation scenario. Both must bind port 53, so they cannot run at the same time.

**Decision**: Split by host type—

| Host type | Local DNS | Rationale |
|----------|---------|------|
| server / k8s | CoreDNS (`modules/services/coredns.nix`)| K8s pods access host DNS through the cni0 bridge; CoreDNS's `forward . <cni0IP>` chain is verified (ADR-010) |
| workstation | Numa | No K8s dependency; Numa provides `.numa` TLD local domains + reverse proxy, more convenient for development |

**Numa does not conflict with K8s**: Numa is installed only on workstation, server keeps CoreDNS. They are independent services on different hosts and do not share port 53.

**NixOS configuration** (`modules/services/numa.nix`):

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

**Key configuration points**:

- `DNSStubListener=no` — must be turned off, otherwise systemd-resolved seizes port 53 and conflicts with Numa
- `AmbientCapabilities = [ "CAP_NET_BIND_SERVICE" ]` — Numa needs this capability to bind privileged ports (53/80/443/853)
- `pkgs.writeText` — the configuration file is stored in read-only `/nix/store`, suitable for purely declarative management
- If you rely on the Numa REST API to update the configuration dynamically, change `ExecStart` to point to the writable path `/var/lib/numa/numa.toml`

**How to introduce it**: Import `modules/services/numa.nix` in `presets/workstation-base.nix`; the server preset does not import it.
