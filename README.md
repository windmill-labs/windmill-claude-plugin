# Windmill Claude Code Plugin

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) for [Windmill](https://www.windmill.dev/) development.

It adds Windmill-specific skills and guidance so Claude Code can work with:
- Windmill scripts across supported runtimes and query languages
- flows, triggers, schedules, resources, and raw apps
- Windmill CLI conventions and generated platform guidance

The plugin content is generated from the source-of-truth system prompts in the main Windmill repo.

## Installation

Add the marketplace:

```
/plugin marketplace add windmill-labs/windmill-claude-plugin
```

Install the plugin:

```
/plugin install windmill-code-plugin@windmill
```
