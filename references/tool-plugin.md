# Tool Plugin Implementation Guide

## Overview

Tool plugins make external APIs usable as nodes in Dify workflows, chatflows, and agent conversations. A tool plugin consists of a **provider** (credentials + metadata) and one or more **tools** (individual operations).

## File Structure

```
my-tool-plugin/
├── provider/
│   └── my_provider.yaml        # Provider definition (credentials, identity)
├── tools/
│   ├── search.yaml             # Tool parameter definition
│   ├── search.py               # Tool implementation
│   ├── translate.yaml
│   └── translate.py
├── _assets/
│   ├── icon.svg
│   └── icon-dark.svg
├── manifest.yaml
├── main.py
└── requirements.txt
```

## Provider YAML

Defines the tool provider's identity and credential fields:

```yaml
identity:
  author: your-name
  name: my_provider               # Provider identifier
  label:
    en_US: My Provider
    zh_Hans: 我的提供商
  description:
    en_US: Provides search and translation tools
  icon: icon.svg
credentials_for_provider:
  api_key:
    type: secret-input
    required: true
    label:
      en_US: API Key
    placeholder:
      en_US: Enter your API key
    help:
      en_US: Get your key from https://example.com/keys
  base_url:
    type: text-input
    required: false
    label:
      en_US: Base URL
    default: "https://api.example.com"
```

## Tool YAML

Defines a single tool's identity, parameters, and output schema:

```yaml
identity:
  name: search                    # Tool identifier
  author: your-name
  label:
    en_US: Search
    zh_Hans: 搜索
  icon: icon.svg
description:
  human:
    en_US: Search for information using keywords
  llm: Search for relevant information based on a query string. Use when the user asks for facts, data, or current information.
parameters:
  - name: query
    type: string                  # string | number | boolean | select | file | files
    required: true
    label:
      en_US: Search Query
    human_description:
      en_US: The search keywords
    llm_description: The query string to search for. Be specific and concise.
    form: llm                     # "llm" = AI infers value; "form" = user presets value

  - name: max_results
    type: number
    required: false
    label:
      en_US: Max Results
    default: 10
    min: 1
    max: 100
    form: form                    # User sets this in the UI, not AI

  - name: language
    type: select
    required: false
    label:
      en_US: Language
    options:
      - value: en
        label:
          en_US: English
      - value: zh
        label:
          en_US: Chinese
    default: en
    form: form
```

### Parameter Types

| Type | Description |
|------|-------------|
| `string` | Text input |
| `number` | Numeric input (supports `min`, `max`) |
| `boolean` | True/false toggle |
| `select` | Dropdown (requires `options`) |
| `file` | Single file upload |
| `files` | Multiple file uploads |

### form Field

- `llm` — The AI model decides the parameter value based on context. Requires `llm_description`.
- `form` — The user configures this in the Dify UI when adding the tool to a workflow. Uses `default` if set.

## Tool Python Implementation

```python
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
import requests


class SearchTool(Tool):
    def _invoke(
        self, user_id: str, tool_parameters: dict
    ) -> ToolInvokeMessage | list[ToolInvokeMessage]:
        # ── Read parameters ──
        query = tool_parameters.get("query", "")
        max_results = tool_parameters.get("max_results", 10)

        # ── Read provider credentials ──
        api_key = self.runtime.credentials.get("api_key", "")
        base_url = self.runtime.credentials.get("base_url", "https://api.example.com")

        # ── Call external API ──
        try:
            resp = requests.get(
                f"{base_url}/search",
                headers={"Authorization": f"Bearer {api_key}"},
                params={"q": query, "limit": max_results},
                timeout=30,
            )
            resp.raise_for_status()
            results = resp.json()
        except requests.RequestException as e:
            return self.create_text_message(f"Search failed: {str(e)}")

        # ── Return results ──
        # For workflow consumption, prefer create_json_message
        return [
            self.create_json_message(results),
            self.create_text_message(
                "\n".join(f"- {r['title']}: {r['url']}" for r in results.get("items", []))
            ),
        ]
```

### Accessing Credentials

Provider credentials are available via `self.runtime.credentials`:

```python
api_key = self.runtime.credentials.get("api_key")
base_url = self.runtime.credentials.get("base_url", "default_value")
```

### Message Factory Methods

| Method | Use Case |
|--------|----------|
| `self.create_text_message(text)` | Human-readable text output |
| `self.create_json_message(dict)` | Structured data for workflow nodes to consume |
| `self.create_blob_message(bytes, meta)` | Binary files; `meta` can include `mime_type` |
| `self.create_image_message(url)` | Image by URL |
| `self.create_link_message(url)` | Clickable hyperlink |
| `self.create_file_message(...)` | File reference |
| `self.create_variable_message(...)` | Variable output |

Return a single message or a list. When returning both JSON and text, the JSON feeds downstream nodes while the text is displayed to users.

## Tool Plugin with Endpoints

A tool plugin can also expose endpoints. Add the endpoint files and reference them in `manifest.yaml`:

```yaml
# manifest.yaml
plugins:
  tools:
    - tools/my_tool.yaml
  endpoints:
    - group/my-plugin.yaml
```

## manifest.yaml for Tool Plugin

```yaml
version: 0.0.1
type: plugin
author: your-name
name: my-tool-plugin
label:
  en_US: My Tool Plugin
description:
  en_US: Search and translation tools
icon: icon.svg
resource:
  memory: 268435456
  permission:
    tool:
      enabled: false          # This is for USING tools, not PROVIDING them
    model:
      enabled: false
    endpoint:
      enabled: false          # Set true only if you also expose endpoints
    app:
      enabled: false
    storage:
      enabled: false
plugins:
  tools:
    - provider/my_provider.yaml
meta:
  version: 0.0.1
  arch: [amd64, arm64]
  runner:
    language: python
    version: "3.12"
    entrypoint: main
created_at: 2025-01-01T00:00:00+08:00
privacy: PRIVACY.md
```
