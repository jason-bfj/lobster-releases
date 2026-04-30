# Lobster CLI

A CLI tool that connects to the Figma MCP server, extracts structured design data from Figma files, and outputs JSON consumable by downstream modules (WordPress block builders, HubSpot generators, Next.js components, etc.).

Runs entirely locally. No backend, no account required beyond a Figma token.

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
```

#### `--workspace` explained

When set, Lobster scans the named Figma section group (default: `"Workspace"`) to find which components are actually used on your page designs, then only extracts those from the library. Also writes a `page-manifest.json` showing every block on every workspace page — flagged as matched or unmatched. Requires `FIGMA_TOKEN`.

---

### `debug`

Call a Figma MCP tool directly and dump the raw response.

```bash
lobster debug <tool> <fileKey> [--node-id <id>]
```

| Tool | Description |
|---|---|
| `get_design_context` | Full design context for a file or node |
| `get_metadata` | File/node metadata (frames, pages, sizes) |
| `get_variable_defs` | Figma variable definitions (design tokens) |
| `get_screenshot` | Node screenshot |

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

Lobster also performs a passive background check after every command and notifies you if an update is available.

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

---

## Output Structure

```
[output-dir]/
  style-guide/
    index.json              ← design tokens (colors, typography, spacing, effects)
  components/
    {slug}/
      index.json            ← assembled block definition
      assets.json           ← image/icon references (only if section has assets)
  page-manifest.json        ← workspace page compositions (only with --workspace)
  [fileKey]/
    _raw/                   ← internal data — useful for debugging
    manifest.json           ← extraction run summary
```

---

## Figma MCP Connection

| Mode | Endpoint | When to use |
|---|---|---|
| Desktop | `http://127.0.0.1:3845/mcp` | Figma desktop app is running — no token needed |
| Remote | `https://mcp.figma.com/mcp` | Headless use — requires `FIGMA_TOKEN` |

Lobster tries the desktop endpoint first. If the Figma app is not running it falls back to the remote endpoint. Override with `--endpoint` on any command.

---

## Self-Update

```bash
lobster update
```

Downloads the latest binary for your platform and replaces your current installation. Lobster also checks passively in the background after every command.

---

## Gotchas

- **Always quote Figma URLs** — the `&` character in URLs is a shell background operator when unquoted
- **`--workspace` requires `FIGMA_TOKEN`** — it needs REST API access to scan page children
- **`--sections` only works in single-page mode** — ignored when extracting multiple pages
- macOS binaries are not code-signed — users may need to allow them in **System Settings → Privacy & Security** on first run



```
Last login: Wed Apr 29 15:01:59 on ttys000
jasonmitchell@MacBookPro ~ % npm uninstall lobster

up to date in 696ms
jasonmitchell@MacBookPro ~ % lober -v
zsh: command not found: lober
jasonmitchell@MacBookPro ~ % lobster -v
[dotenv@17.3.1] injecting env (0) from .env -- tip: ⚙️  suppress all logs with { quiet: true }
error: unknown option '-v'
jasonmitchell@MacBookPro ~ % curl -L https://github.com/jason-bfj/lobster-releases/releases/latest/download/lobster-macos-x64 \
  -o /usr/local/bin/lobster && chmod +x /usr/local/bin/lobster
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 48.9M  100 48.9M    0     0  7347k      0  0:00:06  0:00:06 --:--:-- 9932k
jasonmitchell@MacBookPro ~ % lobster -h
(node:8981) Warning: To load an ES module, set "type": "module" in the package.json or use the .mjs extension.
(Use `lobster --trace-warnings ...` to show where the warning was created)
/snapshot/lobster/node_modules/chalk/source/vendor/supports-color/index.js:1
import process from 'node:process';
^^^^^^

SyntaxError: Cannot use import statement outside a module
    at wrapSafe (node:internal/modules/cjs/loader:1464:18)
    at Module._compile (node:internal/modules/cjs/loader:1495:20)
    at Module._compile (pkg/prelude/bootstrap.js:1926:32)
    at Module._extensions..js (node:internal/modules/cjs/loader:1623:10)
    at Module.load (node:internal/modules/cjs/loader:1266:32)
    at Module._load (node:internal/modules/cjs/loader:1091:12)
    at Module.require (node:internal/modules/cjs/loader:1289:19)
    at Module.require (pkg/prelude/bootstrap.js:1840:31)
    at require (node:internal/modules/helpers:182:18)
    at Object.<anonymous> (/snapshot/lobster/node_modules/chalk/source/index.js:47:37)

Node.js v20.20.2
jasonmitchell@MacBookPro ~ % lobster -help
(node:8983) Warning: To load an ES module, set "type": "module" in the package.json or use the .mjs extension.
(Use `lobster --trace-warnings ...` to show where the warning was created)
/snapshot/lobster/node_modules/chalk/source/vendor/supports-color/index.js:1
import process from 'node:process';
^^^^^^

SyntaxError: Cannot use import statement outside a module
    at wrapSafe (node:internal/modules/cjs/loader:1464:18)
    at Module._compile (node:internal/modules/cjs/loader:1495:20)
    at Module._compile (pkg/prelude/bootstrap.js:1926:32)
    at Module._extensions..js (node:internal/modules/cjs/loader:1623:10)
    at Module.load (node:internal/modules/cjs/loader:1266:32)
    at Module._load (node:internal/modules/cjs/loader:1091:12)
    at Module.require (node:internal/modules/cjs/loader:1289:19)
    at Module.require (pkg/prelude/bootstrap.js:1840:31)
    at require (node:internal/modules/helpers:182:18)
    at Object.<anonymous> (/snapshot/lobster/node_modules/chalk/source/index.js:47:37)

Node.js v20.20.2
jasonmitchell@MacBookPro ~ % lobster extract
(node:8984) Warning: To load an ES module, set "type": "module" in the package.json or use the .mjs extension.
(Use `lobster --trace-warnings ...` to show where the warning was created)
/snapshot/lobster/node_modules/chalk/source/vendor/supports-color/index.js:1
import process from 'node:process';
^^^^^^

SyntaxError: Cannot use import statement outside a module
    at wrapSafe (node:internal/modules/cjs/loader:1464:18)
    at Module._compile (node:internal/modules/cjs/loader:1495:20)
    at Module._compile (pkg/prelude/bootstrap.js:1926:32)
    at Module._extensions..js (node:internal/modules/cjs/loader:1623:10)
    at Module.load (node:internal/modules/cjs/loader:1266:32)
    at Module._load (node:internal/modules/cjs/loader:1091:12)
    at Module.require (node:internal/modules/cjs/loader:1289:19)
    at Module.require (pkg/prelude/bootstrap.js:1840:31)
    at require (node:internal/modules/helpers:182:18)
    at Object.<anonymous> (/snapshot/lobster/node_modules/chalk/source/index.js:47:37)

Node.js v20.20.2
jasonmitchell@MacBookPro ~ % npm install chalk@4

added 6 packages, and audited 7 packages in 2s

2 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
jasonmitchell@MacBookPro ~ % lobster -help      
(node:9049) Warning: To load an ES module, set "type": "module" in the package.json or use the .mjs extension.
(Use `lobster --trace-warnings ...` to show where the warning was created)
/snapshot/lobster/node_modules/chalk/source/vendor/supports-color/index.js:1
import process from 'node:process';
^^^^^^

SyntaxError: Cannot use import statement outside a module
    at wrapSafe (node:internal/modules/cjs/loader:1464:18)
    at Module._compile (node:internal/modules/cjs/loader:1495:20)
    at Module._compile (pkg/prelude/bootstrap.js:1926:32)
    at Module._extensions..js (node:internal/modules/cjs/loader:1623:10)
    at Module.load (node:internal/modules/cjs/loader:1266:32)
    at Module._load (node:internal/modules/cjs/loader:1091:12)
    at Module.require (node:internal/modules/cjs/loader:1289:19)
    at Module.require (pkg/prelude/bootstrap.js:1840:31)
    at require (node:internal/modules/helpers:182:18)
    at Object.<anonymous> (/snapshot/lobster/node_modules/chalk/source/index.js:47:37)

Node.js v20.20.2
jasonmitchell@MacBookPro ~ % chalk -v
zsh: command not found: chalk
jasonmitchell@MacBookPro ~ % npm uninstall chalk

removed 6 packages, and audited 1 package in 488ms

found 0 vulnerabilities
jasonmitchell@MacBookPro ~ % rm /usr/local/bin/lobster
jasonmitchell@MacBookPro ~ % lobster -h
zsh: command not found: lobster
jasonmitchell@MacBookPro ~ % npm uninstall chalk

up to date, audited 1 package in 424ms

found 0 vulnerabilities
jasonmitchell@MacBookPro ~ % 

```
