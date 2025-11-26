# AI-Assisted Migration

## Process

### Setup

- Create project `target-projects/myapp-jmix` with the same base package as in source project.
- Init git repo and commit all.
- Setup IntelliJ IDEA MCP Server
	- Settings -> Tools -> MCP Server
		- Enable
		- Claude Code -> Auto-Configure
- Run Claude Code in `cuba7-jmix2`
- `/mcp`, check that `jetbrains` is connected

### Init agent

Your task is to migrate a project from CUBA Platform to Jmix.
The source project is located in the `source-projects/myapp`. The empty target Jmix project is created in `target-projects/myapp-jmix`.
Read `AGENTS.md` to understand the project structure, migration approach and rules.
