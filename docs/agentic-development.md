# Agentic development in this repository

Claude Code works in this repository with the Group Analytics setup. Everything that guides it
lives in [ga_analytics_standards](https://vitrolife-group.ghe.com/group-analytics/ga_analytics_standards):
how to start, the workflow and what is blocked are in
[getting-started.md](https://vitrolife-group.ghe.com/group-analytics/ga_analytics_standards/blob/main/docs/getting-started.md).

In this repository:

- `CLAUDE.md` and `.claude/settings.json` connect Claude Code to that setup.
- `.claude/work_areas.conf`: at the start of a session Claude asks whether you work on
  Engineering, Data models or Reports, and blocks edits in the other area's folders.
