# OctoPerf — Claude Code plugins

Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
for the [OctoPerf](https://octoperf.com) load-testing platform.

This repo distributes one plugin (`octoperf`) that:

- registers the hosted **OctoPerf MCP server** (`https://api.octoperf.com/mcp`,
  Streamable HTTP, OAuth 2.1 + PKCE + DCR — no API key);
- installs eight workflow **skills** that drive the MCP tools end-to-end
  (auto-correlation, validation triage, scenario diagnosis, bench-report
  reading, PDF export, real-browser probe, scheduling, async polling);
- ships an `AGENTS.md` reference describing every MCP tool the server
  exposes.

## Install

```text
/plugin marketplace add OctoPerf/octoperf-claude-plugins
/plugin install octoperf@octoperf
```

The first MCP call opens a browser for OAuth login on `api.octoperf.com`.
Revoke at any time from **Account → Connected applications** on OctoPerf.

## What you get

| Component             | Location                                | Purpose                                                            |
|-----------------------|-----------------------------------------|--------------------------------------------------------------------|
| MCP server            | `plugins/octoperf/.mcp.json`            | Tools to manage workspaces, projects, VUs, scenarios, bench results |
| AGENTS.md             | `plugins/octoperf/AGENTS.md`            | Full MCP tool catalogue (also served as `octoperf://templates/agents-md`) |
| Skills                | `plugins/octoperf/skills/*/SKILL.md`    | Workflow recipes invoked on natural-language triggers              |

## Skills

| Skill                              | Triggers on                                                            |
|------------------------------------|------------------------------------------------------------------------|
| `octoperf-auto-correlation`        | "correlate the VU", "fix replay errors", "401/403 on replay"           |
| `octoperf-validation-triage`       | "the validation is red", "lots of errors after import"                 |
| `octoperf-scenario-diagnosis`      | "the load test failed", "why are response times so high"               |
| `octoperf-bench-reports`           | "what's the right tool for this widget", widget metric questions       |
| `octoperf-export-bench-report-pdf` | "export the report as PDF", "print the bench report", "share a PDF"    |
| `octoperf-real-browser-probe`      | "EUM probe", "Playwright probe", "TruClient equivalent"                |
| `octoperf-scheduling`              | "schedule the scenario", "run every weekday at 8am"                    |
| `octoperf-async-polling`           | invoked automatically by any skill that has to poll a `taskId` result  |


## License

Apache-2.0
