# Windmill plugin for Claude Code & Cursor

A plugin for [Windmill](https://www.windmill.dev/) development that works with both [Claude Code](https://docs.claude.com/en/docs/claude-code/plugins) and [Cursor](https://cursor.com/docs/reference/plugins).

It adds Windmill-specific skills and guidance so the assistant can work with:
- Windmill scripts across supported runtimes and query languages
- flows, triggers, schedules, resources, and raw apps
- Windmill CLI conventions and generated platform guidance

The plugin content is generated from the source-of-truth system prompts in the main Windmill repo, and shared by both editors — the same `skills/` directory is consumed by Claude Code (via `.claude-plugin/`) and Cursor (via `.cursor-plugin/`).

## Installation

### Claude Code

Add the marketplace, then install the plugin:

```
/plugin marketplace add https://github.com/windmill-labs/windmill-claude-plugin
/plugin install windmill@windmill
```

### Cursor

Install from the [Cursor Marketplace](https://cursor.com/marketplace) once the plugin is listed. The plugin manifests live in `.cursor-plugin/` and `plugins/windmill/.cursor-plugin/`.
