# Lobster CLI

A CLI tool that connects to the Figma MCP server, extracts structured design data from Figma files, and outputs JSON consumable by downstream modules (WordPress block builders, HubSpot generators, Next.js components, etc.).

Runs entirely locally. No backend, no account required beyond a Figma token.

---

## Table of Contents

- [Installation](#installation)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Commands](#commands)
  - [extract](#extract)
  - [debug](#debug)
  - [tools](#tools)
  - [update](#update)
- [Configuration](#configuration)
- [Output Structure](#output-structure)
- [Figma MCP Connection](#figma-mcp-connection)
- [Self-Update](#self-update)
- [Local Development](#local-development)
- [Release Pipeline](#release-pipeline)
- [Gotchas](#gotchas)

---

## Installation

### macOS (Apple Silicon)
```bash
curl -L https://github.com/jason-bfj/lobster-releases/releases/latest/download/lobster-macos-arm64 \
  -o /usr/local/bin/lobster && chmod +x /usr/local/bin/lobster
```

### macOS (Intel)
```bash
curl -L https://github.com/jason-bfj/lobster-releases/releases/latest/download/lobster-macos-x64 \
  -o /usr/local/bin/lobster && chmod +x /usr/local/bin/lobster
```

### Linux
```bash
curl -L https://github.com/jason-bfj/lobster-releases/releases/latest/download/lobster-linux-x64 \
  -o /usr/local/bin/lobster && chmod +x /usr/local/bin/lobster
```

### Windows
Download `lobster-win-x64.exe` from the [releases page](https://github.com/jason-bfj/lobster-releases/releases/latest).

> **macOS note:** Binaries are not code-signed. On first run macOS may block the binary. Go to **System Settings → Privacy & Security** and click **Allow Anyway**.

---

## Prerequisites

- A Figma file you want to extract from
- Either:
  - **Figma desktop app** running (auto-authenticated, no token needed), or
  - A **Figma personal access token** (required for `--workspace` and multi-page extraction)

---

## Quick Start

```bash
# 1. Set your Figma token (recommended)
echo "FIGMA_TOKEN=your_token_here" > .env

# 2. Extract a Figma file — always quote the URL
lobster extract "https://www.figma.com/design/FILEID/Name?node-id=3-75"

# 3. Find your output
ls ./extraction/
```

---

## Commands

### `extract`

Extract design data from a Figma page or file.

```bash
lobster extract <figma-url-or-file-id> [options]
```

| Flag | Default | Description |
|---|---|---|
| `-o, --output <dir>` | `./extraction` | Output directory |
| `--assets` | `false` | Download image and SVG assets |
| `--sections <indexes>` | `"all"` | Comma-separated section indexes, e.g. `"1,3"` (single-page only) |
| `--endpoint <url>` | Auto-detected | Override the Figma MCP endpoint |
| `--token <token>` | `FIGMA_TOKEN` env | Override the Figma personal access token |
| `--workspace [name]` | `"Workspace"` | Only extract components used on pages in this Figma section group |

**Examples:**

```bash
# Full URL — always quote to prevent shell issues with &
lobster extract "https://www.figma.com/design/FILEID/Name?node-id=3-75"

# Bare file ID
lobster extract hSjgQlB3JKKs94TQqyR7bf

# Custom output directory
lobster extract "https://..." -o ./lobster-output/my-project

# Workspace filter — only extracts components referenced by your workspace pages
lobster extract "https://..." --workspace
lobster extract "https://..." --workspace "Design System"

# Include image and SVG asset downloads
lobster extract "https://..." --assets

# Extract only sections 1 and 3 (single-page mode)
lobster extract "https://..." --sections "1,3"
```

#### `--workspace` explained

When set, Lobster scans the named Figma section group (default: `"Workspace"`) to find which components are actually used on your page designs, then only extracts those components from the library. This avoids pulling the entire component library when you only need a subset.

It also writes a `page-manifest.json` showing every block on every workspace page — flagged as matched (extracted) or unmatched (not found in the library). Requires `FIGMA_TOKEN`.

---

### `debug`

Call a Figma MCP tool directly and dump the raw response. Useful for inspecting what data is available before building extraction logic.

```bash
lobster debug <tool> <fileKey> [options]
```

| Argument | Description |
|---|---|
| `<tool>` | MCP tool name (see below) |
| `<fileKey>` | Figma file key (the `FILEID` portion of the URL) |

| Tool | Description |
|---|---|
| `get_design_context` | Full design context for a file or node |
| `get_metadata` | File/node metadata (frames, pages, sizes) |
| `get_variable_defs` | Figma variable definitions (design tokens) |
| `get_screenshot` | Node screenshot |

```bash
lobster debug get_design_context FILEID
lobster debug get_metadata FILEID --node-id 3-75
lobster debug get_variable_defs FILEID
```

Saves the raw response to `./debug-{tool}-{fileKey}.json`.

---

### `tools`

List all tools available on the connected Figma MCP server.

```bash
lobster tools
```

---

### `update`

Check for a newer version and replace the current binary with the latest release.

```bash
lobster update
```

Lobster also performs a passive background check after every command. If a newer version is available you will be notified automatically.

---

## Configuration

### Figma Token

Create a `.env` file in your working directory (never commit this):

```
FIGMA_TOKEN=your_figma_personal_access_token_here
```

Generate a token: Figma → **Account Settings** → **Personal access tokens** → **Generate new token**.

When creating the token, enable these scopes:

| Scope | Access | Required for |
|---|---|---|
| **File content** | Read only | Reading file structure, pages, and node data |
| **Variables** | Read only | Extracting design tokens (colors, typography, spacing) |

Without a token, Lobster falls back to the Figma desktop app's local MCP connection. The desktop app works for basic extraction but `--workspace`, multi-page enumeration, and token extraction via REST all require a token.

---

### `lobster.config.json`

Place in your working directory to control how Lobster categorises Figma pages:

```json
{
  "categoryMap": {
    "Your Page Name": "style-guide"
  },
  "skipPages": ["Readme", "Changelog", "WIP"]
}
```

**`categoryMap`** — override the auto-detected category for specific page names. Valid values: `style-guide`, `generic-blocks`, `custom-blocks`, `pages`, `unknown`.

**`skipPages`** — page names to skip (case-insensitive substring match). Lobster always skips pages containing `cover` or `thumbnail` by default.

Without this file, Lobster auto-detects the category from the Figma page name.

---

## Output Structure

```
[output-dir]/
  style-guide/
    index.json              ← design tokens (colors, typography, spacing, effects)
                              + any style-guide category components
  components/
    {slug}/
      index.json            ← assembled block definition
      assets.json           ← image/icon references (only if section has assets)
  page-manifest.json        ← workspace page compositions (only with --workspace)
  [fileKey]/
    _raw/                   ← internal data — useful for debugging, not for direct use
      section-map.json
      tokens/
        colors.json
        typography.json
        spacing.json
        effects.json
      sections/
        [slug]/
          normalised.json
          assets.json
          metadata.json
      assets/               ← downloaded files (only with --assets)
    manifest.json           ← extraction run summary
```

### `style-guide/index.json`

```json
{
  "extractedAt": "...",
  "colors": {
    "brand-primary": { "hex": "#FF4040", "rgba": "rgba(255,64,64,1)", "opacity": 1 }
  },
  "typography": {
    "heading-1": { "fontFamily": "Inter", "fontWeight": 700, "fontSize": 48, "lineHeight": 67 }
  },
  "spacing": { "xs": 4, "sm": 8, "md": 16, "lg": 24 },
  "effects": { "radii": { "sm": 4, "md": 8 } }
}
```

### `components/{slug}/index.json`

```json
{
  "id": "4174:110099",
  "name": "Navbar / 1 /",
  "role": "section",
  "bounds": { "width": 1440, "height": 80 },
  "layout": { "mode": "HORIZONTAL", "padding": { "top": 0, "right": 80, "bottom": 0, "left": 80 }, "gap": 0 },
  "fills": [],
  "children": [ ... ],
  "assets": { "images": [], "icons": [] },
  "breakpoints": { "mobile": { ... } }
}
```

### `page-manifest.json`

```json
{
  "extractedAt": "...",
  "pages": [
    {
      "id": "6383:168",
      "name": "Homepage Design",
      "slug": "homepage-design",
      "blocks": [
        { "name": "Navbar / 1 /", "nodeId": "12233:2234", "slug": "navbar-1", "matched": true },
        { "name": "Partnership Block", "nodeId": "12233:9999", "slug": "partnership-block", "matched": false }
      ]
    }
  ]
}
```

---

## Figma MCP Connection

Lobster supports two MCP endpoints and auto-detects which to use:

| Mode | Endpoint | When to use |
|---|---|---|
| Desktop | `http://127.0.0.1:3845/mcp` | Figma desktop app is running — no token needed |
| Remote | `https://mcp.figma.com/mcp` | Headless use — requires `FIGMA_TOKEN` |

Lobster tries the desktop endpoint first. If the Figma app is not running it falls back to the remote endpoint. Override with `--endpoint` on any command.

When `FIGMA_TOKEN` is set, Lobster prefers the Figma REST API for structure and token data to avoid burning the MCP daily rate limit.

---

## Self-Update

```bash
lobster update
```

Downloads the latest binary for your platform from [jason-bfj/lobster-releases](https://github.com/jason-bfj/lobster-releases/releases) and replaces your current installation. Lobster also checks passively in the background after every command.

---

## Local Development

Requirements: Node.js >= 20

```bash
# Install dependencies
npm install

# Build TypeScript → dist/
npm run build

# Watch mode (rebuild on save)
npm run dev

# Link globally for local testing (re-run after every build)
npm install -g . --force

# Compile platform binaries (outputs to bin/)
npm run package
```

---

## Release Pipeline

1. Make your changes on `main`
2. Bump the version in `package.json`
3. Commit and push to `main`
4. Trigger the build:
   ```bash
   git push origin main:releases
   ```
5. GitHub Actions will:
   - Build TypeScript
   - Compile 4 platform binaries via `@yao-pkg/pkg`
   - Publish a release to [jason-bfj/lobster-releases](https://github.com/jason-bfj/lobster-releases) with all binaries attached

Alternatively, trigger a release manually from the GitHub Actions UI and specify the version.

### Required secret

The workflow on `jason-bfj/lobster` requires:

| Secret | Value |
|---|---|
| `RELEASES_PAT` | Fine-grained PAT scoped to `jason-bfj/lobster-releases` with **Contents: Read and write** |

PATs expire after 90 days. To renew: GitHub → Settings → Personal access tokens → Fine-grained tokens → Lobster → **Regenerate**, then update the secret in `jason-bfj/lobster` → Settings → Secrets → Actions.

---

## Gotchas

- **Always quote Figma URLs** — the `&` character in URLs is a shell background operator when unquoted
- **`--workspace` requires `FIGMA_TOKEN`** — it needs REST API access to scan page children
- **`--sections` only works in single-page mode** — ignored when extracting multiple pages
- `bin/` is gitignored — compiled binaries are never committed to source
- `private: true` in `package.json` — this package cannot be published to npm
- `pkg` bytecode warnings during build are harmless
- macOS binaries are not code-signed — users may need to allow them in **System Settings → Privacy & Security** on first run
