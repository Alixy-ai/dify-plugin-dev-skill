# dify-plugin-dev

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for developing [Dify](https://dify.ai/) plugins.

## What it does

This skill gives Claude deep knowledge of the Dify plugin SDK, so it can help you:

- **Scaffold** new plugins from scratch (Extension, Tool, Model, Agent Strategy, Trigger, Data Source)
- **Implement** endpoint handlers, tool providers, and model integrations with correct patterns
- **Configure** `manifest.yaml`, group YAML settings, and endpoint/tool YAML definitions
- **Debug** plugins via remote debug connection
- **Package** plugins into `.difypkg` files
- **Publish** to Dify Marketplace via GitHub Actions

## When it triggers

The skill activates automatically when you mention:

- Dify plugin development (create, scaffold, modify, debug, package, publish)
- `manifest.yaml`, `dify_plugin` SDK, `.difypkg`
- Endpoint, Tool, or Model plugin implementation
- Dify plugin architecture questions

## File structure

```
dify-plugin-dev/
├── SKILL.md                          # Main skill (plugin dev lifecycle guide)
└── references/
    ├── manifest-schema.md            # manifest.yaml complete field reference
    ├── endpoint-plugin.md            # Endpoint plugin patterns & examples
    ├── tool-plugin.md                # Tool plugin patterns & examples
    └── publishing.md                 # Packaging, distribution & GitHub Actions workflow
```

## Installation

### Claude Code CLI

```bash
# Clone to your skills directory
git clone https://github.com/Alixy-ai/dify-plugin-dev-skill.git \
  ~/.claude/skills/dify-plugin-dev
```

### Manual

Copy the entire directory to `~/.claude/skills/dify-plugin-dev/`.

## Usage examples

```
> 帮我创建一个 Dify 插件，集成 OpenWeather API，提供天气查询工具

> 给我的 Dify 插件添加一个新的 endpoint，用来导出应用 DSL

> 如何把我的 Dify 插件发布到 Marketplace？
```

## Benchmark

Tested on 3 eval scenarios (Tool plugin creation, Endpoint addition, Publish guide):

| Metric | With Skill | Without Skill |
|--------|-----------|---------------|
| Assertion pass rate | **95%** | 70% |
| Tool plugin correctness | **100%** (8/8) | 37.5% (3/8) |
| Endpoint addition | **100%** (6/6) | 83% (5/6) |
| Publish guide | 100% (6/6) | 100% (6/6) |

The biggest improvement is in creating plugins from scratch, where the skill prevents common mistakes like wrong permission flags, mismatched versions, incorrect `_invoke` signatures, and invalid YAML schema fields.

## Dify plugin types supported

| Type | Description |
|------|-------------|
| Extension (Endpoint) | Custom HTTP endpoints exposed through Dify |
| Tool | Third-party API integrations usable in workflows |
| Model | AI model provider integrations (LLM, embedding, etc.) |
| Agent Strategy | Custom reasoning logic for Agent nodes |
| Trigger | Webhook-based workflow triggers |
| Data Source | External data fetching |

## Key references

- [Dify Plugin Developer Guide](https://docs.dify.ai/en/plugins/develop-plugins/plugin-developer-guide)
- [Dify Plugin SDK (Python)](https://github.com/langgenius/dify-plugin-sdks)
- [Dify Plugin Daemon / CLI](https://github.com/langgenius/dify-plugin-daemon)
- [Dify Marketplace](https://marketplace.dify.ai/)

## License

MIT
