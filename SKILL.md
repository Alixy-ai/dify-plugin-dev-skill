---
name: dify-plugin-dev
description: >
  Guide for developing Dify plugins (Extension, Tool, Model, Agent Strategy, Trigger, Data Source).
  Use this skill whenever the user wants to create, scaffold, modify, debug, package, or publish a Dify plugin,
  or asks about Dify plugin SDK, manifest.yaml, endpoint/tool implementation, plugin architecture,
  or any Dify plugin development question. Also trigger when the user mentions "dify plugin", "dify extension",
  "dify tool provider", "dify model provider", ".difypkg", or references dify_plugin SDK imports.
---

# Dify Plugin Development Guide

You are an expert Dify plugin developer. This skill covers the full lifecycle: scaffolding, implementing, debugging, packaging, and publishing Dify plugins.

## Quick Orientation

Dify plugins extend the Dify platform with custom capabilities. There are six plugin types:

| Type | What it does | Base Class |
|------|-------------|------------|
| **Extension (Endpoint)** | Exposes custom HTTP endpoints | `Endpoint` |
| **Tool** | Integrates third-party APIs as tools usable in workflows/chatflows | `Tool` |
| **Model** | Adds new AI model providers (LLM, embedding, rerank, TTS, STT) | Per model type |
| **Agent Strategy** | Custom reasoning logic for Agent nodes | `AgentStrategy` |
| **Trigger** | Fires workflows on external events (webhooks) | Provider + Events |
| **Data Source** | Fetches data from external systems | Provider + Datasources |

**Constraint**: A plugin cannot mix Tool + Model capabilities in one package.

Before writing any code, ask the user which plugin type they need. If they describe the use case without naming a type, recommend the appropriate one.

## Project Structure

Every Dify plugin follows this core layout:

```
my-plugin/
├── _assets/
│   ├── icon.svg              # Light theme icon
│   └── icon-dark.svg         # Dark theme icon (optional)
├── main.py                   # Entry point (always the same pattern)
├── manifest.yaml             # Plugin metadata and permissions
├── requirements.txt          # Python dependencies
├── pyproject.toml            # Python project config (optional, for uv/pip)
├── .env.example              # Debug environment template
├── .difyignore               # Excluded files when packaging
├── .python-version           # Python version (e.g., 3.12)
├── PRIVACY.md                # Privacy policy (required for Marketplace)
└── README.md
```

Additional directories depend on plugin type:

- **Endpoint plugin**: `group/` (settings YAML) + `endpoints/` (YAML routes + Python handlers)
- **Tool plugin**: `provider/` (provider YAML) + `tools/` (YAML definitions + Python implementations)
- **Model plugin**: `provider/` + `models/{llm,text_embedding,rerank,tts,speech2text}/`
- **Agent Strategy**: `provider/` + `strategies/`

## main.py (Universal Entrypoint)

Every plugin uses the same entrypoint — do not modify this pattern:

```python
from dify_plugin import Plugin, DifyPluginEnv

plugin = Plugin(DifyPluginEnv(MAX_REQUEST_TIMEOUT=120))

if __name__ == "__main__":
    plugin.run()
```

`MAX_REQUEST_TIMEOUT` is in seconds. Adjust based on expected response time.

## manifest.yaml

The manifest defines the plugin's identity, permissions, and runtime configuration. Read `references/manifest-schema.md` for the complete field reference when you need to fill in or modify a manifest.

Key rules:
- `name`: 1-128 chars, lowercase letters/numbers/hyphens/underscores only
- `author`: Must match GitHub username or Marketplace org name
- `version` and `meta.version` should stay in sync
- Enable only the permissions your plugin actually needs
- `meta.runner.entrypoint` is `main` (refers to `main.py`)
- `created_at` uses RFC 3339 format with timezone

## Building an Endpoint Plugin

Endpoint plugins expose HTTP APIs through Dify. This is the most common type for integrating external services. Read `references/endpoint-plugin.md` for the full implementation guide with patterns and examples.

The three files you need per endpoint:

1. **Group YAML** (`group/my-plugin.yaml`) — defines user-configurable settings and lists all endpoints
2. **Endpoint YAML** (`endpoints/my-endpoint.yaml`) — declares the HTTP route and method
3. **Endpoint Python** (`endpoints/my-endpoint.py`) — implements the handler

### Group YAML Pattern

```yaml
settings:
  - name: base_url
    type: text-input          # text-input | secret-input | select | boolean | app-selector
    required: true
    label:
      en_US: Base URL
      zh_Hans: 基础地址
    placeholder:
      en_US: "e.g. https://api.example.com"
    default: "https://api.example.com"
  - name: api_key
    type: secret-input        # Use secret-input for passwords, tokens, keys
    required: true
    label:
      en_US: API Key
      zh_Hans: API 密钥
endpoints:
  - endpoints/my-endpoint.yaml
```

### Endpoint YAML Pattern

```yaml
path: "/my-plugin/resource"       # URL path (werkzeug syntax, supports <param>)
method: "GET"                     # GET | POST | PUT | DELETE | HEAD | OPTIONS
extra:
  python:
    source: "endpoints/my-endpoint.py"
```

### Endpoint Python Pattern

```python
import json
from typing import Mapping
from werkzeug import Request, Response
from dify_plugin import Endpoint

class MyEndpoint(Endpoint):
    def _invoke(self, r: Request, values: Mapping, settings: Mapping) -> Response:
        # r         — werkzeug Request (r.args, r.get_json(), r.headers)
        # values    — URL path parameters
        # settings  — user-configured values from group YAML
        
        return Response(
            json.dumps({"result": "ok"}, ensure_ascii=False),
            status=200,
            content_type="application/json; charset=utf-8",
        )
```

Rules:
- One class per `.py` file, inheriting from `Endpoint`
- Always use `ensure_ascii=False` for proper Unicode/CJK support
- Always set `content_type="application/json; charset=utf-8"`
- Access settings via `settings["field_name"]` — these come from the group YAML

## Building a Tool Plugin

Tool plugins make external APIs available as workflow nodes and chatflow tools. Read `references/tool-plugin.md` for the full implementation guide.

Quick pattern:

```python
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage

class MyTool(Tool):
    def _invoke(self, user_id: str, tool_parameters: dict) -> ToolInvokeMessage | list[ToolInvokeMessage]:
        query = tool_parameters.get("query", "")
        # ... call external API ...
        return self.create_text_message("Result")
```

Tool message factory methods:
- `self.create_text_message(text)` — plain text
- `self.create_json_message(data)` — structured data (for workflow nodes)
- `self.create_blob_message(blob, meta)` — binary files (images, PDFs)
- `self.create_image_message(url)` — image by URL
- `self.create_link_message(url)` — hyperlink

## Reverse Invocation

Plugins can call back into Dify services via `self.session`:

```python
# Call an LLM
self.session.model.llm.invoke(model_config, prompt_messages, tools, stream)

# Call another tool
self.session.tool.invoke(provider, tool_name, params)

# Call a Dify App
self.session.app.chat.invoke(app_id, inputs, response_mode, conversation_id)
self.session.app.workflow.invoke(...)
self.session.app.completion.invoke(...)
```

These require the corresponding `resource.permission` enabled in `manifest.yaml`.

## Persistent Storage

If `resource.permission.storage.enabled: true`:

```python
self.session.storage.set(key, value_bytes)
self.session.storage.get(key) -> bytes
self.session.storage.delete(key)
```

Use for session state, caching, user preferences. Size limited by `storage.size` in manifest.

## Debugging

1. Copy `.env.example` to `.env`
2. Configure remote debug connection:
   ```
   INSTALL_METHOD=remote
   REMOTE_INSTALL_URL=debug.dify.ai:5003
   REMOTE_INSTALL_KEY=<get-from-dify-ui-plugin-management>
   ```
3. Run: `python -m main`
4. Plugin appears in Dify UI marked as "debugging"
5. Test endpoints/tools directly from Dify

## Packaging and Publishing

Read `references/publishing.md` for detailed steps and the GitHub Actions auto-publish workflow.

Quick reference:

```bash
# Package (run from PARENT directory)
./dify plugin package ./my-plugin
# Output: my-plugin.difypkg

# Install locally: Dify UI > Plugin Management > Install > Upload .difypkg
```

Three distribution channels:
1. **Local file** — upload `.difypkg` via Dify UI
2. **GitHub** — users install via repository URL
3. **Dify Marketplace** — submit PR to `langgenius/dify-plugins`, requires review

## Common Patterns and Best Practices

### Shared Client Module
When multiple endpoints call the same external API, extract common logic into a helper module (e.g., `endpoints/my_client.py`). Include session caching with TTL for authenticated APIs:

```python
import threading, time, requests

_session_cache: dict[str, tuple[requests.Session, float]] = {}
_session_lock = threading.Lock()
_SESSION_TTL = 30 * 60  # 30 minutes

def get_session(settings) -> requests.Session:
    cache_key = f"{settings['base_url']}|{settings['email']}"
    with _session_lock:
        cached = _session_cache.get(cache_key)
        if cached and time.time() - cached[1] < _SESSION_TTL:
            return cached[0]
    session = _do_login(settings)
    with _session_lock:
        _session_cache[cache_key] = (session, time.time())
    return session
```

### Error Handling
Always return proper HTTP status codes with JSON error bodies:
```python
try:
    result = do_something()
except Exception as e:
    return Response(
        json.dumps({"error": str(e)}, ensure_ascii=False),
        status=500,
        content_type="application/json; charset=utf-8",
    )
```

### Security
- Use `secret-input` type for all sensitive settings (passwords, API keys, tokens)
- Never hardcode credentials
- Use `hmac.compare_digest` for timing-safe token comparison
- Validate external input at system boundaries

### Internationalization
All `label`, `description`, and `placeholder` fields support multilingual values. Always include `en_US` as primary:
```yaml
label:
  en_US: Field Name
  zh_Hans: 字段名称
```

### No Local File I/O
Plugins may run in serverless environments (AWS Lambda). Use `self.session.storage` for persistence instead of local filesystem operations.

### One Class Per File
Each endpoint/tool `.py` file should contain exactly one class inheriting from the appropriate base class. The class name can be anything, but convention is `PascalCase` + type suffix (e.g., `ListAppsEndpoint`, `SearchTool`).

## Dependencies

Always include in both `requirements.txt` and `pyproject.toml`:

```
dify_plugin>=0.3.0,<0.5.0
```

Add any additional dependencies your plugin needs (e.g., `requests`, `httpx`). Python version should be 3.12+.
