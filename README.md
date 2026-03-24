# Snaply Builder Plugin

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that generates valid Snaply tenant configuration JSON.

## What it does

Describe what you want to build in plain English and the plugin generates ready-to-push JSON config for:

- **Schema** — database tables, columns, and relationships
- **Functions** — pipeline functions with routes, steps, and logic
- **Cronjobs** — scheduled tasks with cron expressions
- **API Settings** — table permissions, auth, SMTP email, environment variables

## Install

```bash
claude plugin add <repo-url>
```

## Usage

In Claude Code, invoke the skill:

```
/snaply-builder a blog with posts and comments
```

Or describe your intent in conversation — the skill activates automatically when you're working on Snaply backend config.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Snaply CLI](https://github.com/nicholasgasior/snaply-cli) (for pushing generated config)

## License

MIT
