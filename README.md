# Snaply Builder Plugin

Generates valid Snaply tenant configuration JSON from a plain-English description. Packaged as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin, but the underlying `SKILL.md` is plain markdown — drop it into any AI coding agent or IDE that supports custom context files (Cursor, Cline, Continue, Copilot, Aider, etc.).

## What it does

Describe what you want to build in plain English and the plugin generates ready-to-push JSON config for:

- **Schema** — database tables, columns, and relationships
- **Functions** — pipeline functions with routes, steps, and logic
- **Cronjobs** — scheduled tasks with cron expressions
- **API Settings** — table permissions, auth, SMTP email, environment variables

## Install

### Claude Code

```bash
claude plugin marketplace add https://github.com/attisvalentin/snaply-builder-plugin
claude plugin install snaply-builder
```

### Claude.ai

1. Clone or download this repo
2. Zip the `skills/snaply-builder` folder
3. Go to **Customize > Skills > Create skill > Upload a skill**
4. Upload the zip file

### Other agents / IDEs

`skills/snaply-builder/SKILL.md` and `skills/snaply-builder/reference.md` are plain markdown — any tool that lets you load custom context can use them directly:

- **Cursor / Windsurf / Cline** — add `SKILL.md` to your project rules (or `.cursor/rules/`), and point at `reference.md` when generating non-trivial configs.
- **Continue, Aider, Copilot custom instructions** — register the file path as a context source or include its contents in your instructions file.
- **API / SDK calls** — paste `SKILL.md` into the system prompt, or attach it as a document on the request.

## Usage

Invoke the skill directly:

```
/snaply-builder a blog with posts and comments
```

Or describe your intent in conversation — the skill activates automatically when you're working on Snaply backend config.

## Requirements

- Any AI coding agent that can load custom markdown context — [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Claude.ai](https://claude.ai), Cursor, Cline, Continue, Copilot, Aider, etc.
- [Snaply CLI](https://snaply-admin-dev-o5uezc.free.laravel.cloud/docs/cli) (for pushing generated config)

## License

MIT
