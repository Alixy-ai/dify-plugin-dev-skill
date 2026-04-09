# Endpoint Plugin Implementation Guide

## Overview

An Endpoint plugin exposes custom HTTP endpoints through Dify. Users configure settings (API keys, URLs) in the Dify UI, and external clients call the endpoints.

## File Structure

```
my-plugin/
├── group/
│   └── my-plugin.yaml          # Settings + endpoint list
├── endpoints/
│   ├── my-endpoint.yaml        # Route: path + method
│   ├── my-endpoint.py          # Handler implementation
│   ├── another-endpoint.yaml
│   ├── another-endpoint.py
│   └── shared_client.py        # Shared helper module (optional)
├── manifest.yaml               # Must have endpoint.enabled: true
├── main.py
└── requirements.txt
```

## Group YAML — Settings and Endpoint List

The group YAML file lives in `group/` and defines:
1. User-configurable settings (shown in Dify UI)
2. List of endpoints belonging to this group

```yaml
settings:
  - name: base_url
    type: text-input
    required: true
    label:
      en_US: Base URL
      zh_Hans: 基础地址
    placeholder:
      en_US: "e.g. https://api.example.com"
      zh_Hans: "例如 https://api.example.com"
    default: "https://api.example.com"

  - name: api_key
    type: secret-input
    required: true
    label:
      en_US: API Key
      zh_Hans: API 密钥
    placeholder:
      en_US: Enter your API key
      zh_Hans: 请输入你的 API 密钥

  - name: mode
    type: select
    required: false
    label:
      en_US: Mode
    options:
      - value: "simple"
        label:
          en_US: Simple
      - value: "advanced"
        label:
          en_US: Advanced
    default: "simple"

  - name: enable_logging
    type: boolean
    required: false
    label:
      en_US: Enable Logging
    default: false

endpoints:
  - endpoints/list-items.yaml
  - endpoints/get-item.yaml
  - endpoints/create-item.yaml
```

### Setting Types

| Type | Description | Extra Fields |
|------|-------------|-------------|
| `text-input` | Plain text input | `default`, `placeholder` |
| `secret-input` | Encrypted input (passwords, tokens) | `default`, `placeholder` |
| `select` | Dropdown menu | `options: [{value, label}]`, `default` |
| `boolean` | Toggle switch | `default` |
| `app-selector` | Dify app picker | — |

All `label`, `placeholder` fields support multilingual values (`en_US`, `zh_Hans`, etc.).

## Endpoint YAML — Route Definition

Each endpoint needs a YAML file declaring the HTTP route:

```yaml
path: "/my-plugin/items"
method: "GET"
extra:
  python:
    source: "endpoints/list-items.py"
```

### Path Syntax

Uses werkzeug URL rules:
- Static: `/my-plugin/items`
- With parameter: `/my-plugin/items/<item_id>`
- Typed parameter: `/my-plugin/items/<int:item_id>`

### Supported Methods

`HEAD`, `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`

## Endpoint Python — Handler Implementation

```python
"""List all items from external API"""

import json
from typing import Mapping

from werkzeug import Request, Response
from dify_plugin import Endpoint


class ListItemsEndpoint(Endpoint):
    def _invoke(self, r: Request, values: Mapping, settings: Mapping) -> Response:
        # ── 1. Read settings ──
        base_url = settings["base_url"]
        api_key = settings["api_key"]

        # ── 2. Read request parameters ──
        page = r.args.get("page", 1, type=int)          # Query params
        # body = r.get_json(silent=True)                 # JSON body (POST/PUT)
        # item_id = values.get("item_id")                # Path params
        # auth = r.headers.get("Authorization")          # Headers

        # ── 3. Call external API ──
        try:
            import requests
            resp = requests.get(
                f"{base_url}/api/items",
                headers={"Authorization": f"Bearer {api_key}"},
                params={"page": page},
                timeout=30,
            )
            resp.raise_for_status()
            data = resp.json()
        except Exception as e:
            return Response(
                json.dumps({"error": str(e)}, ensure_ascii=False),
                status=502,
                content_type="application/json; charset=utf-8",
            )

        # ── 4. Return response ──
        return Response(
            json.dumps(data, ensure_ascii=False, indent=2),
            status=200,
            content_type="application/json; charset=utf-8",
        )
```

### Handler Signature

```python
def _invoke(self, r: Request, values: Mapping, settings: Mapping) -> Response:
```

| Parameter | Type | Source |
|-----------|------|--------|
| `r` | `werkzeug.Request` | The HTTP request from the caller |
| `values` | `Mapping` | Path parameters extracted from URL pattern |
| `settings` | `Mapping` | User-configured values from group YAML |

### Request Object (`r`)

| Access | Method |
|--------|--------|
| Query params | `r.args.get("key", default, type=int)` |
| JSON body | `r.get_json(silent=True)` |
| Form data | `r.form.get("key")` |
| Headers | `r.headers.get("Authorization")` |
| Raw body | `r.data` |
| Method | `r.method` |
| Full URL | `r.url` |

### Response Construction

Always use `ensure_ascii=False` and explicit charset:

```python
Response(
    json.dumps(data, ensure_ascii=False, indent=2),
    status=200,
    content_type="application/json; charset=utf-8",
)
```

Common status codes:
- `200` — success
- `400` — bad request (invalid params)
- `401` — unauthorized
- `404` — not found
- `500` — internal error
- `502` — upstream API error

## Shared Client Module Pattern

When multiple endpoints call the same API, create a shared helper:

```python
# endpoints/my_client.py

import base64
import hmac
import json
import threading
import time
from typing import Mapping

import requests
from werkzeug import Request, Response


# ── Session Caching ──
_cache: dict[str, tuple[requests.Session, float]] = {}
_lock = threading.Lock()
_TTL = 30 * 60  # 30 minutes


def parse_json(resp: requests.Response):
    """Parse JSON from raw bytes to preserve Unicode"""
    return json.loads(resp.content)


def verify_bearer_token(r: Request, settings: Mapping) -> Response | None:
    """Optional Bearer token check. Returns None if valid, 401 Response if invalid."""
    expected = settings.get("access_token", "")
    if not expected:
        return None
    auth = r.headers.get("Authorization", "")
    token = auth[7:] if auth.startswith("Bearer ") else auth
    if not hmac.compare_digest(token, expected):
        return Response(
            json.dumps({"error": "Unauthorized"}, ensure_ascii=False),
            status=401,
            content_type="application/json; charset=utf-8",
        )
    return None


def create_session(settings: Mapping) -> tuple[requests.Session, str]:
    """Get or create a cached authenticated session"""
    base_url = settings["base_url"].rstrip("/")
    cache_key = f"{base_url}|{settings['email']}"
    with _lock:
        cached = _cache.get(cache_key)
        if cached and time.time() - cached[1] < _TTL:
            return cached[0], base_url
    session = _do_login(base_url, settings["email"], settings["password"])
    with _lock:
        _cache[cache_key] = (session, time.time())
    return session, base_url


def _do_login(base_url: str, email: str, password: str) -> requests.Session:
    """Authenticate and return session with tokens set"""
    session = requests.Session()
    encoded_pw = base64.b64encode(password.encode("utf-8")).decode("utf-8")
    resp = session.post(
        f"{base_url}/console/api/login",
        json={"email": email, "password": encoded_pw},
    )
    resp.raise_for_status()
    data = parse_json(resp)
    token = data.get("data", {}).get("access_token") or data.get("access_token")
    if token:
        session.headers["Authorization"] = f"Bearer {token}"
    return session
```

Then import in each endpoint:

```python
from endpoints.my_client import verify_bearer_token, create_session

class MyEndpoint(Endpoint):
    def _invoke(self, r, values, settings):
        auth_err = verify_bearer_token(r, settings)
        if auth_err:
            return auth_err
        session, base_url = create_session(settings)
        # ... use session ...
```

## Adding a New Endpoint to an Existing Plugin

1. Create `endpoints/new-endpoint.yaml` with path and method
2. Create `endpoints/new-endpoint.py` with handler class
3. Add `endpoints/new-endpoint.yaml` to `group/my-plugin.yaml` endpoints list
4. No changes needed to `manifest.yaml` (group YAML is already referenced)
