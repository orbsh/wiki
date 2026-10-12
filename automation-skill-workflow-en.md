# Turning Operation Automation into Skill Workflows

> **Languages:** [English](automation-skill-workflow-en.md) (primary) · [中文](automation-skill-workflow.md)

## Core Philosophy

Turning browser operations on legacy systems into reusable, schedulable Hermes Skills. This is not "writing a script"; it is **reverse-engineering** a system that has no API documentation and turning it into a scriptable, schedulable interface.

## Workflow Overview

```
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Stage 1: Recording (Playwright capture)                                                       │
│  Playwright controls browser → manual operation → auto-capture API → raw JSON                  │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                ↓
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Stage 2: Identification (pattern extraction)                                                  │
│  Analyze request/response structure → identify API patterns (CRUD, pagination, auth flow)      │
│  → mark dynamic parameters (token, timestamp, CSRF) → generate API pattern doc                 │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                ↓
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Stage 3: Refinement (denoising + auth separation)                                             │
│  Remove browser noise (analytics, tracking, WebSocket)                                         │
│  Auth logic sinks into the framework (out-of-band injection), Skill keeps only business logic  │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                ↓
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Stage 4: Generation (Hermes Skill)                                                            │
│  JSON + pattern doc + Prompt → Hermes → generate Skill                                         │
│  ├─ SKILL.md (description, trigger conditions, usage instructions)                             │
│  └─ scripts/ (execution code, auth injected by framework)                                      │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                ↓
┌────────────────────────────────────────────────────────────────────────────────────────────────┐
│  Stage 5: Production (dual-mode degraded execution)                                            │
│  The orchestrator auto-selects the optimal execution strategy:                                 │
│  ├─ Primary mode: HTTP client (millisecond-level, 0% browser overhead)                         │
│  └─ Fallback mode: Headless browser (simulates human behavior, strong safety net)              │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Stage 1: Recording — Playwright Capture

After running the script, the browser stays open until you close the window or type `exit()` in the F12 console. APIs are captured automatically and saved to `captured_api_logs.json`.

### Core Capture Script (stage1_capture.py)

```python
import os
import json
from playwright.sync_api import sync_playwright

OUTPUT_FILE = "captured_api_logs.json"
captured_data = []

def packet_handler(response):
    """Network request interceptor filter"""
    try:
        if "api" in response.url and response.status == 200:
            request = response.request
            record = {
                "url": response.url,
                "method": request.method,
                "headers": dict(request.headers),
                "payload": request.post_data if request.post_data else None,
                "response_body": response.text() if response.status == 200 else None,
                "response_headers": dict(response.headers),
            }
            captured_data.append(record)
            print(f"[captured]: {response.url} ({request.method})")
    except Exception:
        pass

def start_recording(target_url):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=False, args=["--start-maximized"])
        context = browser.new_context(no_viewport=True)
        page = context.new_page()
        page.on("response", packet_handler)
        page.add_init_script("""
        window.exit = function() {
            window.__playwright_exit__ = true;
            return "exiting...";
        };
        """)
        print(f"Opening: {target_url}")
        print("Operate in browser. Close window or type exit() in F12 to stop.")
        page.goto(target_url)
        try:
            while True:
                if page.is_closed() or page.evaluate("() => window.__playwright_exit__"):
                    break
                page.wait_for_timeout(500)
        except Exception:
            pass
        finally:
            context.close()
            browser.close()
        if captured_data:
            with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
                json.dump(captured_data, f, indent=4, ensure_ascii=False)
            print(f"Saved to: {os.path.abspath(OUTPUT_FILE)}")

if __name__ == "__main__":
    start_recording("https://example.com")
```

---

## Stage 2: Identification — API Pattern Extraction

Extract API patterns from the recorded raw JSON. The key at this step is **identifying which are business APIs and which are noise**.

### Identification Dimensions

| Dimension | Identification Method | Example |
|:---|:---|:---|
| **Business API** | URL contains `/api/`, `/v1/`, `/graphql`, and the response is JSON | `POST /api/order/create` |
| **Auth flow** | Login, token refresh, session management | `POST /api/auth/login` → returns token |
| **Dynamic parameters** | Fields that change on every request (token, timestamp, nonce) | `csrf_token`, `X-Request-ID` |
| **Pagination pattern** | The page/offset/cursor parameters of list endpoints | `?page=1&per_page=20` |
| **Dependency chain** | Dependency relationships between requests (A's response is B's parameter) | login → get token → query orders |

### Output: API Pattern Document

```json
{
  "base_url": "https://legacy-system.example.com",
  "auth_flow": {
    "login_endpoint": "POST /api/auth/login",
    "credentials_fields": ["username", "password"],
    "token_response_path": "data.token",
    "token_header": "Authorization",
    "token_prefix": "Bearer "
  },
  "business_apis": [
    {
      "name": "list_orders",
      "method": "GET",
      "path": "/api/orders",
      "params": ["page", "per_page", "status"],
      "depends_on": ["auth_flow"]
    },
    {
      "name": "create_order",
      "method": "POST",
      "path": "/api/orders",
      "body": ["product_id", "quantity", "address"],
      "dynamic_fields": ["csrf_token"],
      "depends_on": ["auth_flow"]
    }
  ],
  "noise_patterns": [
    "/analytics/*",
    "/tracking/*",
    "wss://realtime.example.com/*",
    "*/favicon.ico",
    "*.js",
    "*.css"
  ]
}
```

---

## Stage 3: Refinement — Denoising + Auth Separation

### Denoising

Filter out non-business requests from the recorded data:

| Noise type | Filter rule |
|:---|:---|
| Static resources | `*.js`, `*.css`, `*.png`, `*.svg` |
| Analytics/tracking | `/analytics/*`, `/tracking/*`, Google Tag Manager |
| WebSocket | `wss://` protocol |
| Health checks | `/health`, `/ping`, `/favicon.ico` |
| CDN resources | Requests to third-party domains |

### Pushing Auth Down to the Framework (Out-of-Band Injection)

**Core principle**: the Skill does not hold credentials; the framework injects them at runtime.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Skill (business logic)                                                  │
│  fn::legacy::list_orders(page=1)                                         │
│  ↓ does not know where username/password/token come from                 │
├──────────────────────────────────────────────────────────────────────────┤
│  Framework layer (auth management)                                       │
│  - Reads credentials from the config table                               │
│  - Automatically manages the token lifecycle (acquire, refresh, expire)  │
│  - Injects into request headers                                          │
│  ↓ The Skill only cares about business parameters                        │
└──────────────────────────────────────────────────────────────────────────┘
```

**SurrealDB implementation**:

```SurrealQL
-- Credentials are stored in the config table, not in the Skill
DEFINE TABLE IF NOT EXISTS legacy_auth SCHEMAFULL;
DEFINE FIELD IF NOT EXISTS system_name ON legacy_auth TYPE string;
DEFINE FIELD IF NOT EXISTS base_url ON legacy_auth TYPE string;
DEFINE FIELD IF NOT EXISTS credentials ON legacy_auth TYPE object;
DEFINE FIELD IF NOT EXISTS token ON legacy_auth TYPE option<string>;
DEFINE FIELD IF NOT EXISTS token_expires ON legacy_auth TYPE option<datetime>;

-- Auth function: automatically manages the token lifecycle
DEFINE FUNCTION fn::legacy::auth($system: string) -> string {
    LET $auth = SELECT * FROM legacy_auth WHERE system_name = $system LIMIT 1;
    IF array::len($auth) == 0 {
        THROW "No auth config for system: " + $system;
    };
    -- Check whether the token is valid
    IF $auth[0].token != NONE AND $auth[0].token_expires > time::now() {
        RETURN $auth[0].token;
    };
    -- Token expired or absent, log in again
    LET $login_response = http::post(
        $auth[0].base_url + "/api/auth/login",
        $auth[0].credentials,
        { "Content-Type": "application/json" }
    );
    LET $new_token = $login_response.data.token;
    -- Update token
    UPDATE $auth[0].id SET
        token = $new_token,
        token_expires = time::now() + duration::from::secs(3600);
    RETURN $new_token;
};
```

**The Skill only calls `fn::legacy::auth()` and never touches credentials**:

```SurrealQL
-- Skill example: query orders (auth handled by the framework)
DEFINE FUNCTION fn::legacy::list_orders(
    $system: string,
    $page: int,
    $per_page: int
) {
    LET $token = fn::legacy::auth($system);
    LET $auth_config = SELECT base_url FROM legacy_auth WHERE system_name = $system LIMIT 1;
    RETURN http::get(
        $auth_config[0].base_url + "/api/orders?page=" + string::limit($page) + "&per_page=" + string::limit($per_page),
        { "Authorization": "Bearer " + $token }
    );
};
```

---

## Stage 4: Generation — Hermes Skill

### The Standard Prompt for Hermes

**Role**: reverse engineer and automation expert.

**Task**: based on the API pattern document, generate a Hermes Skill.

**Requirements**:
1. Generate `SKILL.md` (description, trigger conditions, usage instructions)
2. Generate the execution code under `scripts/`
3. **Auth is injected by the framework** — the Skill receives only business parameters and does not handle credentials
4. **Primary mode**: HTTP client (millisecond-level)
5. **Fallback mode**: Headless browser as a safety net

**Input**:
```
[API pattern document JSON]
[refined request/response samples]
```

### Skill Output Structure

```
skills/legacy-system-automation/
├── SKILL.md                    # description, trigger conditions, usage instructions
└── scripts/
    ├── orchestrator.py         # main entry (dual-mode degraded)
    ├── http_client.py          # primary mode: pure HTTP
    └── headless_fallback.py    # fallback mode: Playwright
```

---

## Stage 5: Production — Dual-Mode Degraded Execution

The execution strategy in the production environment is the same as the original plan:

```python
def hermes_orchestrator(*args, **kwargs):
    """Unified entry point, auth injected by the framework"""
    # Auth is handled by SurrealDB fn::legacy::auth()
    # The Skill only cares about business logic
    try:
        return run_with_http_client(*args)
    except Exception:
        return run_with_headless_browser(*args)
```

---

## Advantages of the Approach

| Dimension | Value |
|:---|:---|
| **Reverse engineering** | No API docs needed; Playwright recording + AI identification = automatic extraction of the interface spec |
| **Auth security** | Credentials are managed in the config table and the Skill holds no keys, supporting rotation and auditing |
| **Denoising & refinement** | Automatically filters non-business requests; the Skill contains only valuable APIs |
| **Dual-mode degradation** | HTTP primary mode + browser fallback; failure of a single mode does not interrupt business |
| **Schedulable** | What is generated is a Hermes Skill, which can be triggered on a schedule by cron |

## Cross-References

- **[AI-Friendly Infrastructure](ai-friendly-infrastructure-en.md)**: the positioning of the browser as a reverse-engineering tool.
- **[Unified Data Layer](unified-data-layer-en.md)**: the auth function is stored in a SurrealDB config table, consistent with the computation-pushdown principle.
- **[Agent Compound Interest](agent-compound-interest-en.md)**: the reverse-engineering results of each legacy system settle into a Skill and are reused across projects.
