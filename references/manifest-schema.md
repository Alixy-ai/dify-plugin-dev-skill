# manifest.yaml Complete Schema Reference

## Full Template

```yaml
# ─── Identity ───
version: 0.0.1                          # Semver: major.minor.patch
type: plugin                            # "plugin" or "bundle"
author: your-github-username            # Must match GitHub/Marketplace org
name: my-plugin-name                    # 1-128 chars: lowercase, numbers, hyphens, underscores
label:
  en_US: My Plugin Name                 # Display name (English required)
  zh_Hans: 我的插件                      # Additional languages optional
description:
  en_US: What this plugin does          # Short description
  zh_Hans: 这个插件的功能
icon: icon.svg                          # Relative to _assets/ or project root
icon_dark: icon-dark.svg                # Dark theme variant (optional)

# ─── Resources & Permissions ───
resource:
  memory: 268435456                     # Max memory in bytes (256MB = 268435456)
  permission:
    tool:
      enabled: false                    # Can use tool capabilities
    model:
      enabled: false                    # Can use model capabilities
      llm: false
      text_embedding: false
      rerank: false
      tts: false
      speech2text: false
      moderation: false
    endpoint:
      enabled: true                     # Can expose HTTP endpoints
    app:
      enabled: false                    # Can access Dify App data
    storage:
      enabled: false                    # Can use persistent KV storage
      size: 1024                        # Storage size limit in KB

# ─── Plugin Definitions ───
plugins:
  tools:                                # Tool provider YAML files
    - tools/my_tool.yaml
  models:                               # Model provider YAML files
    - models/my_model.yaml
  endpoints:                            # Endpoint group YAML files
    - group/my-plugin.yaml

# ─── Runtime Metadata ───
meta:
  version: 0.0.1                        # Keep in sync with top-level version
  arch:
    - amd64
    - arm64
  runner:
    language: python                    # Only "python" supported currently
    version: "3.12"                     # Python version
    entrypoint: main                    # Module name (main.py → "main")
  minimum_dify_version: null            # Min Dify version required (null = any)

# ─── Publishing Metadata ───
created_at: 2025-01-01T00:00:00+08:00  # RFC3339 with timezone
privacy: PRIVACY.md                     # Required for Marketplace
verified: false                         # Set by Dify team after review
```

## Field Details

### version
Semver format. Bump for each release. Both `version` and `meta.version` must match.

### type
- `plugin` — a single plugin package (`.difypkg`)
- `bundle` — aggregates multiple plugins (`.difybndl`)

### author
Must match your GitHub username or Marketplace organization name. Used for publishing workflow.

### name
- 1-128 characters
- Only: lowercase letters, numbers, hyphens (`-`), underscores (`_`)
- Must be unique on Marketplace

### resource.memory
Maximum memory allocation in bytes. Common values:
- 128MB = `134217728`
- 256MB = `268435456`
- 512MB = `536870912`

### resource.permission
Enable only what your plugin needs. Each permission unlocks corresponding `self.session.*` APIs:
- `tool.enabled` → `self.session.tool.invoke()`
- `model.enabled` → `self.session.model.llm.invoke()` etc.
- `endpoint.enabled` → HTTP endpoint handlers
- `app.enabled` → `self.session.app.chat.invoke()` etc.
- `storage.enabled` → `self.session.storage.{get,set,delete}()`

### plugins section
Lists paths to YAML definition files. Only include the sections your plugin uses:
- `tools:` — for Tool plugins
- `models:` — for Model plugins
- `endpoints:` — for Endpoint/Extension plugins

### meta.runner
- `language`: Only `python` is currently supported
- `version`: Python version string (use `"3.12"`)
- `entrypoint`: Module name without `.py` extension

## Permissions by Plugin Type

| Plugin Type | endpoint | tool | model | app | storage |
|------------|----------|------|-------|-----|---------|
| Extension (Endpoint only) | **true** | false | false | false | optional |
| Tool | optional | false | false | false | optional |
| Tool + Endpoint | **true** | false | false | false | optional |
| Model | false | false | **true** | false | optional |
| Agent Strategy | false | **true** | **true** | false | optional |

## Restrictions
- Cannot enable both `tool` and `model` permissions in one plugin
- Must have at least one capability (tools, models, or endpoints)
- `storage.size` is in KB, only relevant when `storage.enabled: true`
