# Jmix AI-Assisted Migration (CUBA Platform 7.2 → Jmix 2)

This repository is designed to facilitate migration of application projects from CUBA Platform to Jmix with the help of AI agents.

See also [jmix-framework/jmix-migration-from-v1](https://github.com/jmix-framework/jmix-migration-from-v1)

Below we assume you are using Claude Code, but other agents should work as well.

## Migration process

### Setup

- Create `workspace/source-projects` folder and copy or clone the source CUBA project into it, for example `source-projects/myapp`.
- Create project `workspace/target-projects/myapp-jmix` using Jmix Studio, with the same base package as in source project.
- Init git repo and commit all.
- Setup IntelliJ IDEA MCP Server
	- Settings -> Tools -> MCP Server
		- Enable
		- Claude Code -> Auto-Configure
- Open both `myapp` and `myapp-jmix` projects in IntelliJ to be accessible by MCP server.
- Open terminal in `workspace` and run Claude Code.
- `/mcp`, check that `jetbrains` is connected.

### Sequence

We recommend to proceed with migration in the following sequence: entities, fetch plans, business logic, fragments, screens. If the project is large, split each phase to the smaller steps, e.g. by packages.

### Example prompts

After starting the agent and before giving any migration commands, initialize it by the following prompt (replace `myapp` with your app folder name):

```
Your task is to migrate a project from CUBA Platform to Jmix.
The source project is located in the `source-projects/myapp`. The target Jmix project is created in `target-projects/myapp-jmix`.
Read `AGENTS.md` to understand the project structure, migration approach and rules.
```

**Migration prompts:**

```
Migrate all entities.
```

```
Migrate all shared fetch plans.
```

```
Migrate all business logic.
```

```
Migrate fragments if any.
```

```
Migrate all screens in `com.company.myapp.gui.sample` package.
```

```
Migrate all screens.
```
