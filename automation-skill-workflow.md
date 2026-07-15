# 操作自动化转 Skill 工作流

## 核心理念

将遗留系统的浏览器操作转化为可复用、可定时执行的 Hermes Skill。不是"写脚本"，是**逆向工程**一个没有 API 文档的系统，将其转化为可脚本化、可调度的接口。

## 工作流总览

```
┌─────────────────────────────────────────────────────────────────┐
│  阶段一：录制（Playwright 抓包）                                   │
│  Playwright 控制浏览器 → 人工操作 → 自动捕获 API → 原始 JSON       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  阶段二：识别（模式提取）                                          │
│  分析请求/响应结构 → 识别 API 模式（CRUD、分页、认证流程）            │
│  → 标记动态参数（token、时间戳、CSRF）→ 生成 API 模式文档           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  阶段三：提纯（去噪 + 认证分离）                                    │
│  去掉浏览器噪声（analytics、追踪、WebSocket）                      │
│  认证逻辑下沉到框架（带外注入），Skill 只保留业务逻辑                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  阶段四：生成（Hermes Skill）                                     │
│  JSON + 模式文档 + Prompt → Hermes → 生成 Skill                  │
│  ├─ SKILL.md（描述、触发条件、使用说明）                            │
│  └─ scripts/（执行代码，认证由框架注入）                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  阶段五：生产（双模降级执行）                                       │
│  编排器自动选择最优执行策略：                                       │
│  ├─ 主模式：HTTP 客户端（毫秒级，0% 浏览器开销）                    │
│  └─ 备用模式：Headless 浏览器（仿真人类行为，强力兜底）             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 阶段一：录制——Playwright 抓包

运行脚本后，浏览器保持打开状态，直到关闭窗口或在 F12 控制台输入 `exit()`。自动捕获 API 并保存到 `captured_api_logs.json`。

### 核心抓包脚本 (stage1_capture.py)

```python
import os
import json
from playwright.sync_api import sync_playwright

OUTPUT_FILE = "captured_api_logs.json"
captured_data = []

def packet_handler(response):
    """网络请求拦截过滤器"""
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

## 阶段二：识别——API 模式提取

从录制的原始 JSON 中提取 API 模式。这一步的关键是**识别哪些是业务 API，哪些是噪声**。

### 识别维度

| 维度 | 识别方法 | 示例 |
|:---|:---|:---|
| **业务 API** | URL 包含 `/api/`、`/v1/`、`/graphql`，响应为 JSON | `POST /api/order/create` |
| **认证流程** | 登录、token 刷新、session 管理 | `POST /api/auth/login` → 返回 token |
| **动态参数** | 每次请求变化的字段（token、时间戳、nonce） | `csrf_token`、`X-Request-ID` |
| **分页模式** | 列表接口的 page/offset/cursor 参数 | `?page=1&per_page=20` |
| **依赖链** | 请求之间的依赖关系（A 的响应是 B 的参数） | 登录 → 获取 token → 查询订单 |

### 输出：API 模式文档

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

## 阶段三：提纯——去噪 + 认证分离

### 去噪

从录制数据中过滤掉非业务请求：

| 噪声类型 | 过滤规则 |
|:---|:---|
| 静态资源 | `*.js`, `*.css`, `*.png`, `*.svg` |
| 分析追踪 | `/analytics/*`, `/tracking/*`, Google Tag Manager |
| WebSocket | `wss://` 协议 |
| 健康检查 | `/health`, `/ping`, `/favicon.ico` |
| CDN 资源 | 第三方域名的请求 |

### 认证下沉到框架（带外注入）

**核心原则**：Skill 不持有凭证，凭证由框架在运行时注入。

```
┌──────────────────────────────────────────────────┐
│  Skill（业务逻辑）                                │
│  fn::legacy::list_orders(page=1)                 │
│  ↓ 不知道 username/password/token 从哪来          │
├──────────────────────────────────────────────────┤
│  框架层（认证管理）                                │
│  - 读取 config 表中的凭证                         │
│  - 自动管理 token 生命周期（获取、刷新、过期）      │
│  - 注入到请求 headers                             │
│  ↓ Skill 只关心业务参数                           │
└──────────────────────────────────────────────────┘
```

**SurrealDB 实现**：

```surql
-- 凭证存储在 config 表，不在 Skill 里
DEFINE TABLE IF NOT EXISTS legacy_auth SCHEMAFULL;
DEFINE FIELD IF NOT EXISTS system_name ON legacy_auth TYPE string;
DEFINE FIELD IF NOT EXISTS base_url ON legacy_auth TYPE string;
DEFINE FIELD IF NOT EXISTS credentials ON legacy_auth TYPE object;
DEFINE FIELD IF NOT EXISTS token ON legacy_auth TYPE option<string>;
DEFINE FIELD IF NOT EXISTS token_expires ON legacy_auth TYPE option<datetime>;

-- 认证函数：自动管理 token 生命周期
DEFINE FUNCTION fn::legacy::auth($system: string) -> string {
    LET $auth = SELECT * FROM legacy_auth WHERE system_name = $system LIMIT 1;
    IF array::len($auth) == 0 {
        THROW "No auth config for system: " + $system;
    };
    -- 检查 token 是否有效
    IF $auth[0].token != NONE AND $auth[0].token_expires > time::now() {
        RETURN $auth[0].token;
    };
    -- Token 过期或不存在，重新登录
    LET $login_response = http::post(
        $auth[0].base_url + "/api/auth/login",
        $auth[0].credentials,
        { "Content-Type": "application/json" }
    );
    LET $new_token = $login_response.data.token;
    -- 更新 token
    UPDATE $auth[0].id SET
        token = $new_token,
        token_expires = time::now() + duration::from::secs(3600);
    RETURN $new_token;
};
```

**Skill 只调用 `fn::legacy::auth()`，不接触凭证**：

```surql
-- Skill 示例：查询订单（认证由框架处理）
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

## 阶段四：生成——Hermes Skill

### 给 Hermes 的标准 Prompt

**角色**：逆向工程师与自动化专家。

**任务**：根据 API 模式文档，生成一个 Hermes Skill。

**要求**：
1. 生成 `SKILL.md`（描述、触发条件、使用说明）
2. 生成 `scripts/` 下的执行代码
3. **认证由框架注入**——Skill 只接收业务参数，不处理凭证
4. **主模式**：HTTP 客户端（毫秒级）
5. **备用模式**：Headless 浏览器兜底

**输入**：
```
[API 模式文档 JSON]
[提纯后的请求/响应样本]
```

### Skill 输出结构

```
skills/legacy-system-automation/
├── SKILL.md                    # 描述、触发条件、使用说明
└── scripts/
    ├── orchestrator.py         # 主入口（双模降级）
    ├── http_client.py          # 主模式：纯 HTTP
    └── headless_fallback.py    # 备用模式：Playwright
```

---

## 阶段五：生产——双模降级执行

生产环境的执行策略与原方案一致：

```python
def hermes_orchestrator(*args, **kwargs):
    """统一入口，认证由框架注入"""
    # 认证由 SurrealDB fn::legacy::auth() 处理
    # Skill 只关心业务逻辑
    try:
        return run_with_http_client(*args)
    except Exception:
        return run_with_headless_browser(*args)
```

---

## 方案优势

| 维度 | 价值 |
|:---|:---|
| **逆向工程** | 不需要 API 文档，Playwright 录制 + AI 识别 = 自动提取接口规范 |
| **认证安全** | 凭证在 config 表中管理，Skill 不持有密钥，支持轮换和审计 |
| **去噪提纯** | 自动过滤非业务请求，Skill 只包含有价值的 API |
| **双模降级** | HTTP 主模式 + 浏览器备用，单一模式失效不中断业务 |
| **可调度** | 生成的是 Hermes Skill，可被 cron 定时触发 |

## 交叉引用

- **[AI 友好基础设施](ai-friendly-infrastructure.md)**：浏览器作为逆向工程工具的定位。
- **[统一数据层](unified-data-layer.md)**：认证函数存储在 SurrealDB config 表中，与计算下推原则一致。
- **[Agent 复利](agent-compound-interest.md)**：每个遗留系统的逆向工程成果沉淀为 Skill，跨项目复用。
