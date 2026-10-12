# Aura + Fluxora DevOps: Script-as-Code Engineering Practice

> **Languages:** [English](aura-fluxora-devops-en.md) (primary) · [中文](aura-fluxora-devops.md)

**Status**: Architecture design complete
**Date**: 2026-07-06
**Core philosophy**: Script-as-Code — embedded scripts are treated on equal footing with Rust code and brought under Git management

---

## Overview

After Fluxora migrated from Kafka to Aura, business logic changed from standalone Rust services into embedded scripts (Steel/PyO3/Wasm). This brought a whole new set of engineering management challenges:

**Core questions**:
- How are scripts version-controlled?
- How are they tested?
- How are they deployed?
- How are they rolled back?
- How are they audited?

**Solution**: Script-as-Code — treat embedded scripts as first-class citizens, bring them into the Git repository, and manage them automatically through the `set()` API.

→ See also [Aura Architecture](aura-architecture-en.md), [Fluxora Architecture](projects/fluxora-architecture-en.md)

---

## 1. Architecture Design: Script-as-Code

### 1.1 Script Classification

| Type | Storage Location | Update Method | Use Case |
|------|---------|---------|---------|
| **Core logic** | Compiled into the binary | Redeploy | Stable foundational functionality |
| **Dynamic policies** | Fjall `booth_defs` partition | Hot deployment via the `set()` API | Business rules that change frequently |
| **User scripts** | Uploaded by the user | Web UI | Custom extensions |

### 1.2 Script Persistence: Fjall Storage

Scripts are no longer read from the filesystem; instead they are stored in Fjall's `booth_defs` partition. The booth definition itself is persistent, replicable state:

```
fjall partition: "booth_defs"
  key:   <booth_type_name>
  value: CBOR { lang, script_bytes, version, content_hash, committed_at, committed_by }
```

- **Versioning**: Fjall's LSM-Tree naturally supports versioning; each `set()` retains a new version, and older versions can be rolled back
- **Deduplication**: `set()` computes the hash of `script_bytes` before committing and compares it with the latest version — if identical, it is ignored and no new version is written
- **Cross-node sync**: Scripts are deployed independently together with the nodes (a meta instance per node + git/S3 static asset distribution); there is no need to manually place script files on every node
- **Instance activation**: The `on()` handler reads the latest script version from Fjall when a booth instance is activated and loads it into the corresponding VM for execution; after the instance is evicted, the next activation reads it again

### 1.3 Endpoint Discovery: the `interface_schema()` Convention

**No manifest file (YAML manifest) is required**. Each script exports `interface_schema()`, which returns the receives/emits declarations:

```python
def interface_schema():
    return {
        "receives": {
            "add_to_cart": {
                "key": "user_id",
                "params": { "type": "object", "properties": { ... } },
                "returns": { "type": "object", "properties": { ... } }  # optional
            }
        },
        "emits": {
            "cart_updated": {
                "schema": { "type": "object", "properties": { ... } }
            }
        }
    }
```

- **Script as documentation**: A glance at `interface_schema()` tells you what events this booth receives and what events it emits
- **Route table construction**: On startup the Host calls `interface_schema()` and builds the event routing table (event name → partition key → JSON Schema → booth definition → on handler)
- **Hot reload entry point**: Script modified → reloaded → `interface_schema()` called again → route table updated, with no need to recompile the Rust host
- **Rust booths**: The `#[aura::schema]` macro generates them automatically at compile time; no hand-writing required

### 1.4 The `set()` API: Hot Deployment

```python
# Commit or update a booth definition at runtime
set("order_booth", lang="python", script=script_bytes)
```

What `set()` defines is a **type**, not an instance. Booth instances are activated on demand by the Arena according to the partition key. Calling `set()` at runtime hot-swaps the booth implementation — not only its behavior, but also its language.

### 1.5 Architecture Topology

```
[Fluxora project repository]
├── ui/                    # UI layer (Brick DSL)
│   ├── templates/         # UI templates
│   └── components/        # Component definitions
├── scripts/               # Business logic scripts
│   ├── chat/              # Chat business
│   │   └── on_message.py  # Python script (contains interface_schema())
│   ├── crm/               # CRM business
│   │   └── on_lead.scm    # Steel Lisp script
│   └── analysis/          # Analysis business
│       └── on_event.wasm  # Wasm plugin
├── tests/                 # Script tests
│   ├── chat_test.rs       # Uses ArenaHarness
│   └── crm_test.rs
└── .github/workflows/     # CI/CD
    └── deploy-scripts.yml
```

---

## 2. GitOps Workflow

### 2.1 The Complete Flow

```
[Developer modifies a script]
    ↓
[git commit]
    ↓
[GitHub Actions CI]
    ├── 1. Syntax check (linter per language)
    ├── 2. Unit tests (ArenaHarness sandbox)
    ├── 3. Integration tests (simulated Aura environment)
    └── 4. Call the set() API to deploy the script
    ↓
[PR review]
    ↓
[Merge to main]
    ↓
[ArgoCD automatic deployment]
    ├── 1. Pull the latest script
    ├── 2. Call the Aura set() API
    ├── 3. Aura validates the interface_schema() structure
    ├── 4. Script written to Fjall (versioned)
    └── 5. Each node deploys independently (git/S3 distributes script assets)
    ↓
[Aura runs the new script (hot reload)]
```

### 2.2 CI/CD Configuration Example

```yaml
# .github/workflows/deploy-scripts.yml
name: Deploy Scripts to Aura

on:
  push:
    branches: [main]
    paths: ['scripts/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Scripts
        run: |
          # Steel Lisp scripts
          steel-lint scripts/**/*.scm
          # Python scripts
          ruff check scripts/**/*.py

      - name: Run Script Tests
        run: |
          cargo test --features aura-test

      - name: Deploy to Aura
        run: |
          # Iterate over the scripts/ directory and call the set() API one by one
          for f in scripts/**/*.py; do
            booth_name=$(basename "$f" .py)
            auractl arena set "$booth_name" python "$f"
          done
          for f in scripts/**/*.scm; do
            booth_name=$(basename "$f" .scm)
            auractl arena set "$booth_name" steel "$f"
          done
```

### 2.3 ArgoCD Configuration

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - aura-deployment.yaml

configMapGenerator:
  - name: aura-config
    files:
      - config.toml

# Different environments use different script versions (controlled via the Aura API)
patches:
  - target:
      kind: Deployment
      name: aura
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: "scripts-v1.2.3"  # production environment
```

---

## 3. Script Testing

### 3.1 The ArenaHarness Test Framework

```rust
#[cfg(test)]
mod tests {
    use aura::test::ArenaHarness;

    #[aura::test]
    async fn test_chat_on_message() {
        let mut harness = ArenaHarness::new();

        // Load the script
        harness.set("chat_booth", "python", include_bytes!("../scripts/chat/on_message.py"));

        // Deliver an event
        harness.emit("on_message", serde_json::json!({
            "user_id": "user-001",
            "message": "Hello"
        }));

        // Verify the emitted event
        let emitted = harness.collect_emits("send_response");
        assert_eq!(emitted.len(), 1);
        assert!(emitted[0]["response"].as_str().unwrap().contains("Hello"));
    }
}
```

### 3.2 Multi-language Testing

ArenaHarness does not care about the script language — it loads scripts via `set()`, delivers events via `emit()`, and verifies results via `collect_emits()`. The same test can cover booths written in Python, Steel Lisp, or Wasm.

---

## 4. Script Dependency Management

### 4.1 Shared Library Structure

```
scripts/
├── lib/                 # Shared library
│   ├── utils.py         # Python utility functions
│   └── db.scm           # Steel Lisp database operations
├── chat/
│   └── on_message.py    # import lib.utils
└── crm/
    └── on_lead.scm      # (import "lib/db")
```

### 4.2 Dependency Imports

```python
# scripts/chat/on_message.py
from lib.utils import log_info, send_response

def interface_schema():
    return { "receives": { ... }, "emits": { ... } }

def on_message(ctx, event):
    user_id = event["user_id"]
    log_info(f"Message from {user_id}")
    ctx.emit("send_response", {"user_id": user_id, "text": "Hello!"})
    ctx.commit()
```

### 4.3 Dependency Resolution

When a script is deployed to Fjall via `set()`, the Host parses the import statements and recursively loads the dependencies into Fjall. During cross-node sync, the dependency relationships are replicated along with it.

---

## 5. Script Debugging

### 5.1 Local Development

```bash
# Single-file startup — no Docker, no etcd, no database required
$ aura dev order_booth.py

# Output:
# [Aura] booth order_booth started
# [Aura] realm: default
# [Aura] State: Fjall local mode (./data/booth/)
# [Aura] Hot reload: watches for file changes, edits take effect immediately
```

File changes automatically trigger a reload → `interface_schema()` called again → route table updated.

### 5.2 Tracing API

```bash
# Enable tracing
curl -X POST https://aura.example.com/api/debug/trace \
  -d '{"booth": "chat_booth", "enabled": true}'

# Fetch the execution log
curl https://aura.example.com/api/debug/log?booth=chat_booth
```

---

## 6. Script Rollback

### 6.1 Version Management

Fjall's LSM-Tree retains the version of every `set()`. A rollback means re-running `set()` with an older version number:

```bash
# auractl rollback
auractl arena rollback chat_booth --to-version 3

# Or via the API
curl -X POST https://aura.example.com/api/booth/rollback \
  -d '{"booth": "chat_booth", "version": 3}'
```

### 6.2 Version History

```bash
# View the version history
auractl arena history chat_booth

# Output:
# version 5  python  2026-07-06T15:30:00Z  deployed_by=alice  git=abc123
# version 4  python  2026-07-06T14:20:00Z  deployed_by=bob    git=def456
# version 3  steel   2026-07-05T10:00:00Z  deployed_by=alice  git=789abc
```

---

## 7. Audit Log

### 7.1 Change Records

Every `set()` in Fjall automatically records metadata (`committed_at`, `committed_by`, `version`). Combined with the Git commit SHA, this forms a complete audit chain:

```bash
# Query the script change history
curl https://aura.example.com/api/audit/booth/chat_booth

# Example response
{
  "versions": [
    {
      "version": 5,
      "lang": "python",
      "committed_at": "2026-07-06T15:30:00Z",
      "committed_by": "alice@example.com",
      "git_commit": "abc123def456"
    },
    {
      "version": 4,
      "lang": "python",
      "committed_at": "2026-07-06T14:20:00Z",
      "committed_by": "bob@example.com",
      "git_commit": "def456ghi789"
    }
  ]
}
```

---

## 8. Best Practices

### 8.1 Branch Strategy

```
main          ← production
  ↓
staging       ← staging environment
  ↓
feature/*     ← feature branch
```

### 8.2 Environment Isolation

| Environment | Script Version | Deployment Method |
|------|---------|---------|
| **Development** | feature branch | `aura dev` local startup |
| **Testing** | staging branch | CI automatically calls `set()` |
| **Production** | main branch | ArgoCD automatically calls `set()` |

### 8.3 Script Quality Gate

```yaml
# .github/workflows/quality-gate.yml
name: Script Quality Gate

on:
  pull_request:
    paths: ['scripts/**']

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint
        run: |
          steel-lint scripts/**/*.scm
          ruff check scripts/**/*.py

      - name: Test
        run: cargo test --features aura-test

      - name: Validate interface_schema
        run: |
          # Load the script and validate that interface_schema() returns a valid structure
          auractl arena validate scripts/**/*.py
          auractl arena validate scripts/**/*.scm
```

### 8.4 Script Performance Monitoring

```rust
// Aura script performance metrics
pub struct ScriptMetrics {
    pub execution_time: Duration,
    pub memory_usage: usize,
    pub error_count: u64,
}

impl ScriptMetrics {
    pub fn record(&self, booth_name: &str) {
        metrics::histogram!("script_execution_time", self.execution_time.as_millis() as f64, "booth" => booth_name);
        metrics::gauge!("script_memory_usage", self.memory_usage as f64, "booth" => booth_name);
        metrics::counter!("script_errors", self.error_count, "booth" => booth_name);
    }
}
```

---

## 9. Engineering Management Challenges and Solutions

| Challenge | Solution |
|------|---------|
| **Version control** | Scripts live in the Git repository and are versioned by commit SHA; Fjall retains the runtime versions |
| **Testing** | ArenaHarness sandbox tests + CI automation |
| **Dependency management** | Shared library directory + import mechanism; recursively loaded into Fjall at deployment time |
| **Debugging** | `aura dev` file-watching hot reload + tracing API |
| **Rollback** | Fjall LSM-Tree versioning; `auractl arena rollback` specifies an older version |
| **GitOps** | ArgoCD automatically calls `set()` + environment isolation |
| **Audit** | Dual tracking via Fjall metadata + Git commit SHA |
| **Cross-node sync** | Script assets distributed via git/S3; no manual deployment of script files |
| **Endpoint discovery** | the `interface_schema()` convention; no YAML manifest file required |
| **Performance** | Prometheus/Grafana monitor script execution metrics |

---

## 10. Key Design Principles

1. **Script-as-Code**: scripts and Rust code are treated on equal footing and brought under Git management
2. **CI/CD automation**: every script change must pass CI tests
3. **Hot reload**: `set()` API hot deployment, `interface_schema()` hot route-table updates, no restart required
4. **Audit trail**: dual recording via Fjall version metadata + Git commit SHA
5. **Environment isolation**: different environments use different script versions
6. **Quality gate**: scripts must pass lint, tests, and `interface_schema()` validation
7. **Performance monitoring**: real-time monitoring of script execution time and memory usage
8. **No manifest file**: the `interface_schema()` convention replaces the YAML manifest; scripts self-describe their endpoints
9. **Storage as asset**: scripts are stored in Fjall (a meta instance per node + git/S3 distribution); no manual deployment required

---

## Cross-references

This document is the complete design of Aura + Fluxora DevOps, and together with the following detailed analyses it forms a complete decision closure loop:

- **[Aura Architecture](aura-architecture-en.md)**: a modern distributed booth engine with storage-compute integration.
- **[Fluxora Architecture](projects/fluxora-architecture-en.md)**: an event-driven UI framework, including the path for a future migration to Aura.
- **[Arrow Unified HTAP Engine](arrow-unified-htap-engine-en.md)**: Fjall + Arrow + Polars full-chain storage-compute integration.

**The unified first principles**: no technology worship, no swallowing the grand promises open source paints, look only at the real physical limits of hardware and at team productivity.
