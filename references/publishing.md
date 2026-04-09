# Packaging and Publishing Guide

## Packaging

The Dify CLI tool (`dify-plugin-daemon`) packages plugins into `.difypkg` files.

### Install the CLI

Download from [dify-plugin-daemon releases](https://github.com/langgenius/dify-plugin-daemon/releases):

```bash
# macOS (Apple Silicon)
wget https://github.com/langgenius/dify-plugin-daemon/releases/download/0.0.6/dify-plugin-darwin-arm64
chmod +x dify-plugin-darwin-arm64
mv dify-plugin-darwin-arm64 /usr/local/bin/dify-plugin

# macOS (Intel)
wget https://github.com/langgenius/dify-plugin-daemon/releases/download/0.0.6/dify-plugin-darwin-amd64

# Linux (amd64)
wget https://github.com/langgenius/dify-plugin-daemon/releases/download/0.0.6/dify-plugin-linux-amd64
```

### Package Command

Run from the **parent directory** of your plugin:

```bash
dify-plugin plugin package ./my-plugin
# Output: my-plugin-0.0.1.difypkg
```

With custom output name:

```bash
dify-plugin plugin package ./my-plugin -o my-plugin-v1.difypkg
```

### .difyignore

Exclude files from the package (same syntax as `.gitignore`):

```
__pycache__/
*.py[cod]
*.egg-info/
.git/
.github/
.env
.venv/
*.difypkg
.DS_Store
.idea/
.vscode/
```

## Local Installation

1. Open Dify UI → Plugin Management
2. Click "Install Plugin" → "Via Local File"
3. Upload the `.difypkg` file (or drag-and-drop)
4. Configure settings when prompted

## GitHub Distribution

Host your plugin in a GitHub repository. Users install via:
- Dify UI → Plugin Management → Install from GitHub
- Provide the repository URL

No review process required. Users trust the source.

## Dify Marketplace Publishing

### Prerequisites

1. Fork the `langgenius/dify-plugins` repository on GitHub
2. Create a GitHub Personal Access Token with `repo` scope
3. Add the token as a repository secret named `PLUGIN_ACTION`
4. Set `author` in `manifest.yaml` to your GitHub username
5. Include `PRIVACY.md` in your plugin

### Marketplace Requirements

- Plugin must be unique (no duplicates of existing plugins)
- English (`en_US`) must be the primary language for all user-facing text
- API keys and credentials must not be hardcoded
- Must include privacy policy (`PRIVACY.md`)
- Must pass automated validation

### Manual Submission

1. Package your plugin
2. Create a branch in your fork of `langgenius/dify-plugins`
3. Add your `.difypkg` and metadata
4. Create a PR to `langgenius/dify-plugins:main`
5. Wait for review and merge

### Automated GitHub Actions Workflow

Add this workflow to your plugin repo at `.github/workflows/plugin-publish.yml`:

```yaml
name: Publish Dify Plugin

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Download CLI tool
        run: |
          mkdir -p $RUNNER_TEMP/bin
          cd $RUNNER_TEMP/bin
          wget https://github.com/langgenius/dify-plugin-daemon/releases/download/0.0.6/dify-plugin-linux-amd64
          chmod +x dify-plugin-linux-amd64

      - name: Get plugin info from manifest
        id: info
        run: |
          PLUGIN_NAME=$(grep "^name:" manifest.yaml | cut -d' ' -f2)
          VERSION=$(grep "^version:" manifest.yaml | cut -d' ' -f2)
          AUTHOR=$(grep "^author:" manifest.yaml | cut -d' ' -f2)
          echo "plugin_name=$PLUGIN_NAME" >> $GITHUB_OUTPUT
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "author=$AUTHOR" >> $GITHUB_OUTPUT

      - name: Package plugin
        run: |
          PACKAGE="${{ steps.info.outputs.plugin_name }}-${{ steps.info.outputs.version }}.difypkg"
          $RUNNER_TEMP/bin/dify-plugin-linux-amd64 plugin package . -o "$PACKAGE"

      - name: Checkout dify-plugins repo
        uses: actions/checkout@v3
        with:
          repository: ${{ steps.info.outputs.author }}/dify-plugins
          path: dify-plugins
          token: ${{ secrets.PLUGIN_ACTION }}

      - name: Copy package and create branch
        run: |
          cd dify-plugins
          PLUGIN_NAME="${{ steps.info.outputs.plugin_name }}"
          VERSION="${{ steps.info.outputs.version }}"
          BRANCH="bump-${PLUGIN_NAME}-plugin-${VERSION}"
          
          mkdir -p "plugins/${{ steps.info.outputs.author }}/$PLUGIN_NAME"
          cp "$GITHUB_WORKSPACE/${PLUGIN_NAME}-${VERSION}.difypkg" \
             "plugins/${{ steps.info.outputs.author }}/$PLUGIN_NAME/"
          
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git checkout -b "$BRANCH"
          git add .
          git commit -m "bump $PLUGIN_NAME plugin to version $VERSION"
          git push -u origin "$BRANCH" --force

      - name: Create PR
        env:
          GH_TOKEN: ${{ secrets.PLUGIN_ACTION }}
        run: |
          PLUGIN_NAME="${{ steps.info.outputs.plugin_name }}"
          VERSION="${{ steps.info.outputs.version }}"
          AUTHOR="${{ steps.info.outputs.author }}"
          BRANCH="bump-${PLUGIN_NAME}-plugin-${VERSION}"
          
          gh pr create \
            --repo langgenius/dify-plugins \
            --head "${AUTHOR}:${BRANCH}" \
            --base main \
            --title "bump ${PLUGIN_NAME} plugin to version ${VERSION}" \
            --body "Automated plugin update from ${AUTHOR}/${PLUGIN_NAME} release ${VERSION}"
```

### Publishing Workflow

1. Bump `version` and `meta.version` in `manifest.yaml`
2. Commit and push
3. Create a GitHub Release (tag matching version)
4. Workflow automatically:
   - Packages the plugin
   - Pushes to your fork of `dify-plugins`
   - Creates a PR to `langgenius/dify-plugins`
5. Dify team reviews and merges

## Bundle Plugins

Bundles aggregate multiple plugins for batch installation:

```yaml
# manifest.yaml (bundle)
version: 0.0.1
type: bundle
plugins:
  - source: marketplace
    id: author/plugin-name
    version: 0.0.1
  - source: github
    id: github.com/user/repo
    version: 0.0.1
    release: "v1.0.0"
  - source: package
    path: plugins/embedded-plugin.difypkg
```

Package: `dify-plugin bundle package ./my-bundle` → produces `.difybndl` file.

## Scaffolding a New Plugin

```bash
dify-plugin plugin init
```

Interactive prompts:
1. Plugin name (1-128 chars)
2. Author name
3. Description
4. Plugin type: tool / model / extension / agent-strategy
5. Multilingual README (y/n)

Generates the complete project structure with boilerplate files.
