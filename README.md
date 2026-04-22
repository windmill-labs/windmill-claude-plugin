# Windmill Claude Code Plugin

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) for [Windmill](https://www.windmill.dev/) development. Provides skills for writing scripts, flows, triggers, schedules, apps, and more — directly from Claude Code.

## Installation

Add the marketplace:

```
/plugin marketplace add windmill-labs/windmill-claude-plugin
```

Install the plugin:

```
/plugin install windmill-code-plugin@windmill
```

## Skills

### Scripts

Write Windmill scripts in any supported language:

- **Python** — `/write-script-python3`
- **TypeScript (Bun)** — `/write-script-bun`
- **TypeScript (Deno)** — `/write-script-deno`
- **TypeScript (Native)** — `/write-script-nativets`
- **Bun Native** — `/write-script-bunnative`
- **Go** — `/write-script-go`
- **Bash** — `/write-script-bash`
- **Rust** — `/write-script-rust`
- **PHP** — `/write-script-php`
- **C#** — `/write-script-csharp`
- **Java** — `/write-script-java`
- **PowerShell** — `/write-script-powershell`
- **R** — `/write-script-rlang`

### Queries

- **PostgreSQL** — `/write-script-postgresql`
- **MySQL** — `/write-script-mysql`
- **MS SQL** — `/write-script-mssql`
- **BigQuery** — `/write-script-bigquery`
- **Snowflake** — `/write-script-snowflake`
- **DuckDB** — `/write-script-duckdb`
- **GraphQL** — `/write-script-graphql`

### Workflows & Infrastructure

- **Flows** — `/write-flow`
- **Triggers** — `/triggers`
- **Schedules** — `/schedules`
- **Resources** — `/resources`
- **Raw Apps** — `/raw-app`
- **CLI Commands** — `/cli-commands`
