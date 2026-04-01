# Marketplace & Plugin Schema Reference

## marketplace.json

Located at: `.claude-plugin/marketplace.json` (repository root)

### Required fields

| Field | Type | Description |
|---|---|---|
| `name` | string | Marketplace identifier in kebab-case. Public-facing name users see when installing. |
| `owner` | object | Marketplace maintainer info. |
| `owner.name` | string | Name of the maintainer or team. |
| `plugins` | array | List of available plugins. |

### Optional fields

| Field | Type | Description |
|---|---|---|
| `owner.email` | string | Contact email for the maintainer. |
| `metadata.description` | string | Brief marketplace description. |
| `metadata.version` | string | Marketplace version. |
| `metadata.pluginRoot` | string | Base directory prepended to relative plugin source paths. E.g., `"./plugins"` lets you write `"source": "formatter"` instead of `"source": "./plugins/formatter"`. |

### Plugin entry fields

Each item in `plugins` array:

**Required:**

| Field | Type | Description |
|---|---|---|
| `name` | string | Plugin identifier in kebab-case. Determines skill namespace (`/plugin-name:skill-name`). |
| `source` | string or object | Where to fetch the plugin from (see Source Types below). |

**Optional:**

| Field | Type | Description |
|---|---|---|
| `description` | string | Brief plugin description. |
| `version` | string | Semantic version. |
| `author` | object | `{ "name": "...", "email": "..." }` |
| `homepage` | string | Plugin homepage URL. |
| `repository` | string | Source code repository URL. |
| `license` | string | SPDX license identifier (e.g., MIT). |
| `keywords` | array | Tags for discovery. |
| `category` | string | Plugin category. |
| `strict` | boolean | Default `true`. Controls whether `plugin.json` is authority for component definitions. |

---

## Source types

### Relative path (same repo)

```json
"source": "./plugins/my-plugin"
```
- Must start with `./`
- Resolves relative to marketplace root
- Cannot use `../`

### GitHub repository

```json
"source": {
  "source": "github",
  "repo": "owner/plugin-repo",
  "ref": "v2.0.0",
  "sha": "a1b2c3d4..."
}
```
`ref` and `sha` are optional.

### Git URL

```json
"source": {
  "source": "url",
  "url": "https://gitlab.com/team/plugin.git",
  "ref": "main",
  "sha": "a1b2c3d4..."
}
```

### Git subdirectory (monorepo)

```json
"source": {
  "source": "git-subdir",
  "url": "https://github.com/org/monorepo.git",
  "path": "tools/claude-plugin",
  "ref": "v2.0.0",
  "sha": "a1b2c3d4..."
}
```
Uses sparse clone. `url` accepts GitHub shorthand (`owner/repo`).

### npm package

```json
"source": {
  "source": "npm",
  "package": "@acme/claude-plugin",
  "version": "2.1.0",
  "registry": "https://npm.example.com"
}
```
`version` supports ranges like `"^2.0.0"`. `registry` is optional.

---

## plugin.json

Located at: `<plugin-root>/.claude-plugin/plugin.json`

### Required fields

| Field | Type | Description |
|---|---|---|
| `name` | string | Plugin identifier. Must match the name in marketplace.json. |
| `description` | string | What the plugin does. |

### Optional fields

| Field | Type | Description |
|---|---|---|
| `version` | string | Semantic version. |
| `author` | object | `{ "name": "...", "email": "..." }` |
| `homepage` | string | Project URL. |
| `repository` | string | Source repository. |
| `license` | string | License type. |
| `keywords` | array | Discovery tags. |

---

## SKILL.md frontmatter

Located at: `<plugin-root>/skills/<skill-name>/SKILL.md`

```yaml
---
name: skill-name
description: "What the skill does and when to trigger it."
disable-model-invocation: false
---
```

| Field | Type | Description |
|---|---|---|
| `name` | string | Skill identifier. Should match folder name. |
| `description` | string | Primary trigger mechanism. Claude reads this to decide activation. |
| `disable-model-invocation` | boolean | Optional. Set `true` to prevent automatic model invocation (manual `/plugin:skill` only). |

---

## Reserved marketplace names

These names are reserved for Anthropic:
- `claude-code-marketplace`, `claude-code-plugins`, `claude-plugins-official`
- `anthropic-marketplace`, `anthropic-plugins`
- `agent-skills`, `knowledge-work-plugins`, `life-sciences`

---

## Complete example

```json
{
  "name": "company-tools",
  "owner": {
    "name": "DevTools Team",
    "email": "devtools@example.com"
  },
  "metadata": {
    "description": "Company-wide tools and automations",
    "version": "1.0.0",
    "pluginRoot": "./plugins"
  },
  "plugins": [
    {
      "name": "code-formatter",
      "source": "./plugins/formatter",
      "description": "Automatic code formatting",
      "version": "2.1.0",
      "author": {
        "name": "DevTools Team"
      },
      "category": "productivity",
      "keywords": ["formatting", "code-style"]
    },
    {
      "name": "deploy-tools",
      "source": {
        "source": "github",
        "repo": "company/deploy-plugin",
        "ref": "stable"
      },
      "description": "Deployment automation"
    }
  ]
}
```
