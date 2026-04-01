---
name: claude-plugin-marketplace-setup
description: "Guide for creating and organizing Claude Code plugin marketplaces. Use when the user wants to create a new plugin, set up a marketplace, organize multiple skills into plugins, fix marketplace.json or plugin.json schema errors, or publish plugins for distribution. Also trigger when the user encounters 'Invalid schema' or 'Failed to parse marketplace file' errors."
---

# Claude Plugin Marketplace Setup

Create and organize Claude Code plugin marketplaces with correct structure and configuration.

## Quick Overview

A **marketplace** is a repository that distributes one or more **plugins**. Each **plugin** contains one or more **skills** (and optionally commands, agents, hooks, MCP servers).

```
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json        ← marketplace definition
├── plugins/
│   ├── plugin-a/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json     ← plugin manifest
│   │   └── skills/
│   │       └── my-skill/
│   │           └── SKILL.md    ← skill content
│   └── plugin-b/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       └── skills/
│           ├── skill-one/
│           │   └── SKILL.md
│           └── skill-two/
│               └── SKILL.md
└── README.md
```

The `.claude-plugin/` directory only contains JSON config files. All other content (skills, commands, agents) lives at the plugin root level, not inside `.claude-plugin/`.

## Step-by-step workflow

### 1. Plan your plugin organization

Before creating files, decide how to group skills into plugins:

- **Same domain → same plugin.** Skills that share a topic (e.g., all Agent SDK related) belong in one plugin. Users install at the plugin level, so grouping by domain keeps installation clean.
- **Different purposes → different plugins.** A coding tool and a documentation generator serve different needs — separate plugins let users install only what they want.
- One plugin can contain many skills. There's no need to create a separate plugin per skill unless they genuinely serve independent audiences.

### 2. Create marketplace.json

Path: `.claude-plugin/marketplace.json` at the repository root.

Read `references/marketplace-schema.md` for the complete schema with all required and optional fields.

**Minimal working example:**

```json
{
  "name": "my-marketplace",
  "owner": {
    "name": "Your Name"
  },
  "plugins": [
    {
      "name": "my-plugin",
      "description": "What this plugin does",
      "source": "./plugins/my-plugin"
    }
  ]
}
```

**Common mistakes to avoid:**
- Missing the top-level `name` field — this causes "Invalid schema" errors
- Using `"source": "."` instead of `"source": "./"` or `"source": "./plugins/xxx"`
- Source paths must start with `"./"` for relative paths
- Forgetting the `owner` object (it's required, with at least a `name` inside)

### 3. Create plugin.json for each plugin

Path: `plugins/<plugin-name>/.claude-plugin/plugin.json`

```json
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "author": {
    "name": "Your Name"
  }
}
```

The `name` here determines the skill namespace — skills are invoked as `/plugin-name:skill-name`.

### 4. Create skills

Path: `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`

```yaml
---
name: skill-name
description: "When and why to use this skill. Be specific about trigger scenarios."
---

# Skill Title

Instructions for Claude when this skill is activated.
```

The skill folder name and the `name` in frontmatter should match. The `description` in frontmatter is the primary trigger mechanism — Claude reads it to decide whether to activate the skill.

### 5. Install and validate

```bash
# Install locally
claude /plugin /path/to/marketplace

# Or add via GitHub
claude /plugin owner/repo

# Reload after changes
/reload-plugins

# Invoke a skill
/plugin-name:skill-name
```

### 6. Update marketplace.json when adding new plugins

When you add a new plugin directory, add a corresponding entry to the `plugins` array in marketplace.json. Each entry needs at minimum `name`, `description`, and `source`.

## Plugin source types

For plugins within the same repo, use relative paths:
```json
"source": "./plugins/my-plugin"
```

For external plugins, use GitHub shorthand, git URLs, or npm packages. See `references/marketplace-schema.md` for all source types.

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| "Invalid schema: name" | Missing top-level `name` in marketplace.json | Add `"name": "marketplace-name"` |
| "Invalid input: plugins.0.source" | Invalid source path format | Use `"./"` prefix for relative paths |
| "Unknown skill: xxx" | Skill not found or plugin not installed | Check plugin is enabled in settings, run `/reload-plugins` |
| Plugin not showing | Cache stale | Run `/reload-plugins` or reinstall via `/plugin` |

## Reference

For complete schema details including all optional fields, source types, and advanced configuration, read `references/marketplace-schema.md`.
